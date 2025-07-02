// External and local service dependencies (ensure these are correctly available in your project)
const StatesService = require('atm-states');
const ScreensService = require('atm-screens');
const FITsService = require('atm-fits');
const CryptoService = require('../services/crypto.js');
const DisplayService = require('../services/display.js');
const OperationCodeBufferService = require('atm-opcode-buffer');
const Trace = require('atm-trace');
const Pinblock = require('pinblock'); // Used in CryptoService, but listed here in original
const des3 = require('node-cardcrypto').des; // Used in CryptoService, but listed here in original
const ATMHardwareService = require('atm-hardware'); // Assuming this exists

class ATM {
  /**
   * ATM Class Constructor.
   * Initializes ATM components and internal state.
   * @param {object} settings - Configuration settings (from electron-settings).
   * @param {object} log - Logging service.
   * @param {object} ndcParser - Instance of the NdcParser for message parsing and formatting.
   * @param {object} dependencies - Object containing instances/constructors for other services.
   * This is a dynamic way to inject dependencies, making the ATM class more testable and flexible.
   */
  constructor(settings, log, ndcParser, dependencies) {
    this.settings = settings;
    this.log = log;
    this.ndcParser = ndcParser; // Injected NdcParser

    // Initialize core services using provided dependencies
    this.trace = dependencies.Trace ? new dependencies.Trace() : new Trace();
    this.states = dependencies.StatesService ? new dependencies.StatesService(settings, log, this.trace) : new StatesService(settings, log, this.trace);
    this.screens = dependencies.ScreensService ? new dependencies.ScreensService(settings, log, this.trace) : new ScreensService(settings, log, this.trace);
    this.FITs = dependencies.FITsService ? new dependencies.FITsService(settings, log, this.trace) : new FITsService(settings, log, this.trace);
    this.crypto = dependencies.CryptoService ? new dependencies.CryptoService(settings, log) : new CryptoService(settings, log);
    this.display = dependencies.DisplayService ? new dependencies.DisplayService(this.screens, log) : new DisplayService(this.screens, log);
    this.pinblock = dependencies.Pinblock ? new dependencies.Pinblock() : new Pinblock(); // Moved from CryptoService
    this.opcode = dependencies.OperationCodeBufferService ? new dependencies.OperationCodeBufferService() : new OperationCodeBufferService();
    this.hardware = dependencies.ATMHardwareService ? new dependencies.ATMHardwareService() : new ATMHardwareService();

    // Internal ATM state
    this.setStatus('Offline');
    this.initBuffers();
    this.initCounters(); // Will be updated to be dynamic
    this.current_state = null;
    this.buttons_pressed = [];
    this.activeFDKs = [];
    this.transaction_request = null; // Stores request object to be sent
    this.interactive_transaction = false;

    // Dynamic configuration values from host (Enhanced Configuration Data)
    this.hostConfig = {
      hardware_configuration: '157F000901020483000001B1000000010202047F7F00', // Default static, can be overridden
      sensor_status: '000000000000', // Default static, can be overridden
      initial_screen_number: '001', // Default, can be overridden
      // Add other dynamic config parameters here
    };
  }

  /**
   * Retrieves the content of a specified buffer.
   * @param {string} type - The type of buffer ('pin', 'B', 'C', 'opcode', 'amount').
   * @returns {string} The buffer content.
   */
  getBuffer(type){
    switch(type){
      case 'pin': return this.PIN_buffer;
      case 'B': return this.buffer_B;
      case 'C': return this.buffer_C;
      case 'opcode': return this.opcode.getBuffer();
      case 'amount': return this.amount_buffer;
      default:
        this.log.error(`ATM.getBuffer(): Unknown buffer type: ${type}`);
        return '';
    }
  }

  /**
   * Checks whether a given Function Display Key (FDK) button is active.
   * @param {string} button - The FDK button (e.g., 'A', 'G'). Case-insensitive.
   * @returns {boolean} True if FDK is active, false otherwise.
   */
  isFDKButtonActive(button){
    if (!button) return false;
    return this.activeFDKs.includes(button.toUpperCase());
  }

  /**
   * Sets the current FDK active mask.
   * @param {string} mask - 1. Number from 000 to 255 (as string), OR
   * 2. Binary mask (as string, e.g., '100011000').
   */
  setFDKsActiveMask(mask){
    if (!mask || mask.length === 0) {
      this.log.error('Empty FDK mask provided.');
      this.activeFDKs = []; // Clear active FDKs if mask is empty
      return;
    }

    this.activeFDKs = [];
    let FDKs = ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I']; // 'E' included here for binary mask processing

    if (mask.length <= 3 && !isNaN(parseInt(mask, 10))) {
      // Mask is a number (0-255)
      const numericMask = parseInt(mask, 10);
      if (numericMask > 255 || numericMask < 0) {
        this.log.error('Invalid FDK mask number: ' + mask);
        return;
      }
      // FDKs A, B, C, D, F, G, H, I map to bits. Diebold typically skips E.
      // Order: A,B,C,D,F,G,H,I maps to bits 0-7.
      const FDK_MAP = ['A', 'B', 'C', 'D', 'F', 'G', 'H', 'I'];
      for (let bit = 0; bit < 8; bit++) {
        if ((numericMask >> bit) & 1) { // Check if the bit is set
          this.activeFDKs.push(FDK_MAP[bit]);
        }
      }
    } else if (mask.length > 0) {
      // Mask is a binary string (e.g., '100011000')
      // The first character of the mask (index 0) is a 'Numeric Keys activator' and is usually ignored for FDKs.
      // FDKs A-I correspond to subsequent bits.
      // Your original code slices off the first char for binary mask, keeping the same FDKs array for A-I.
      // Let's assume standard A-I mapping for binary string.
      const binaryFDKs = mask.substring(1); // Skip the numeric keys activator bit if present
      for (let i = 0; i < binaryFDKs.length && i < FDKs.length; i++) {
        if (binaryFDKs[i] === '1') {
          this.activeFDKs.push(FDKs[i]);
        }
      }
    } else {
      this.log.error('Invalid FDK mask format: ' + mask);
    }
  }


  /**
   * Generates a Terminal State Reply message data object.
   * This data is used by the builder/parser to form the outgoing NDC message.
   * Values like hardware_configuration are now dynamic from `this.hostConfig`.
   * @param {string} command_code - The command code received from the host (e.g., 'Send Configuration Information').
   * @returns {object} Structured data for a Solicited Status Terminal State reply.
   */
  getTerminalStateReply(command_code){
    let reply = {};
    reply.terminal_command = command_code; // This field is used by builder.js

    switch(command_code){
      case 'Send Configuration Information':
        reply.config_id = this.getConfigID();
        reply.hardware_fitness = this.hardware.getHardwareFitness();
        // Use dynamic hardware_configuration if available, else fallback to default
        reply.hardware_configuration = this.hostConfig.hardware_configuration;
        reply.supplies_status = this.hardware.getSuppliesStatus();
        // Use dynamic sensor_status if available, else fallback to default
        reply.sensor_status = this.hostConfig.sensor_status;
        reply.release_number = this.hardware.getReleaseNumber();
        reply.ndc_software_id = this.hardware.getHarwareID();
        break;

      case 'Send Configuration ID':
        reply.config_id = this.getConfigID();
        break;

      case 'Send Supply Counters':
        Object.assign(reply, this.getSupplyCounters()); // Merge supply counters into reply
        break;

      default:
        this.log.warn(`ATM.getTerminalStateReply(): Unhandled terminal command code: ${command_code}`);
        break;
    }
    return reply;
  }

  /**
   * Creates a structured data object for a Solicited Status message reply.
   * This object will then be formatted by the NdcParser.
   * @param {string} status - The status descriptor ('Ready', 'Command Reject', 'Terminal State', etc.).
   * @param {string} [command_code] - Optional command code if status is 'Terminal State'.
   * @returns {object} Structured data for a Solicited Status message.
   */
  replySolicitedStatus(status, command_code){
    let replyData = {
      messageId: 'ReadyState', // Use the messageId for formatting (maps to "ReadyState" in XML)
      data: {
        LUNO_ATM: this.settings.get('host').luno || '009', // Get ATM's LUNO dynamically
        StatusDescriptor: status === 'Ready' ? '9' : (status === 'Command Reject' ? 'A' : (status === 'Specific Command Reject' ? 'C' : '9')), // Map status to NDC code
      }
    };

    if (status === 'Terminal State') {
      // The 'ReadyState' formatter currently does not directly support embedding Terminal State data.
      // This will need to be handled by a more specific message definition or by extending 'ReadyState' formatter.
      // For now, let's include terminal state data in a nested property and let the Builder adapt or
      // ensure the ReadyState formatter in Parser.js is updated.
      // The current Builder.js seems to expect these fields directly for 'Terminal State' type.
      replyData.data = {
          ...replyData.data, // Keep LUNO_ATM, StatusDescriptor
          ...this.getTerminalStateReply(command_code), // Add terminal state data
          StatusDescriptor: 'F', // As per Builder.js, 'F' for config info, '2' for counters
          // Note: The original Builder maps 'Terminal State' to an 'F' code or '2' for counters, not '9' or 'A'.
          // This requires careful alignment between how ATM prepares and how Builder/Parser formats.
          // For now, let's explicitly set status_descriptor as expected by Builder if it directly processes.
          status_descriptor: status // Keep for Builder's logic if it relies on this.
      };

      // Correcting the StatusDescriptor based on Builder's expectation for Terminal State
      if (command_code === 'Send Configuration Information' || command_code === 'Send Configuration ID') {
          replyData.data.StatusDescriptor = 'F'; // 'F' for config data
      } else if (command_code === 'Send Supply Counters') {
          replyData.data.StatusDescriptor = 'F'; // 'F2' is pattern in builder for counters
          replyData.data.SubStatusDescriptor = '2'; // For specific counter info
      } else {
        replyData.data.StatusDescriptor = 'F'; // Default to F for other terminal states
      }
    }
    return replyData; // Return the structured object
  }

  /**
   * Processes an incoming Terminal Command message from the host.
   * @param {object} data - Parsed Terminal Command message object.
   * @returns {object|null} A structured response message object, or null if no response needed.
   */
  processTerminalCommand(data){
    switch(data.command_code){
      case 'Go in-service':
        this.setStatus('In-Service');
        this.processState(this.hostConfig.initial_screen_number || '000'); // Use dynamic initial screen
        this.initBuffers();
        this.activeFDKs = [];
        return this.replySolicitedStatus('Ready'); // Always reply Ready
      case 'Go out-of-service':
        this.setStatus('Out-Of-Service');
        this.initBuffers();
        this.activeFDKs = [];
        this.card = null;
        return this.replySolicitedStatus('Ready'); // Always reply Ready
      case 'Send Configuration Information':
      case 'Send Configuration ID':
      case 'Send Supply Counters':
        return this.replySolicitedStatus('Terminal State', data.command_code);

      default:
        this.log.error('ATM.processTerminalCommand(): unknown command code: ' + data.command_code);
        return this.replySolicitedStatus('Command Reject');
    }
  }

  /**
   * Processes incoming Data Command messages (Customization Commands).
   * This is where Enhanced Configuration Data will be handled.
   * @param {object} data - Parsed Data Command message object.
   * @returns {object} A structured response message object.
   */
  processCustomizationCommand(data){
    switch(data.message_identifier){
      case 'Screen Data load':
        if(this.screens.add(data.screens))
          return this.replySolicitedStatus('Ready');
        else
          return this.replySolicitedStatus('Command Reject');

      case 'State Tables load':
        if(this.states.add(data.states))
          return this.replySolicitedStatus('Ready');
        else
          return this.replySolicitedStatus('Command Reject');

      case 'FIT Data load':
        if(this.FITs.add(data.FITs))
          return this.replySolicitedStatus('Ready');
        else
          return this.replySolicitedStatus('Command Reject');

      case 'Configuration ID number load':
        if(data.config_id){
          this.setConfigID(data.config_id);
          return this.replySolicitedStatus('Ready');
        } else {
          this.log.info('ATM.processCustomizationCommand(): no Config ID provided');
          return this.replySolicitedStatus('Command Reject');
        }
      
      // *** NEW: Handle Enhanced Configuration Data Load ***
      case 'Enhanced Configuration Data Load':
        if (data.parameters && Array.isArray(data.parameters)) {
          this.log.info('ATM: Processing Enhanced Configuration Data.');
          data.parameters.forEach(param => {
            this.log.info(`  Config Parameter: ID=${param.id}, Value=${param.value}`);
            // Dynamically apply configuration parameters
            // This is a crucial area for customization based on your NDC spec
            switch (param.id) {
              case '000': // Example: Initial Screen Number
                this.hostConfig.initial_screen_number = param.value.padStart(3, '0');
                this.log.info(`  Set initial_screen_number to: ${this.hostConfig.initial_screen_number}`);
                break;
              case '010': // Example: Hardware Configuration String
                this.hostConfig.hardware_configuration = param.value;
                this.log.info(`  Set hardware_configuration to: ${this.hostConfig.hardware_configuration}`);
                break;
              case '020': // Example: Sensor Status
                this.hostConfig.sensor_status = param.value;
                this.log.info(`  Set sensor_status to: ${this.hostConfig.sensor_status}`);
                break;
              // Add more cases for other parameter IDs from your 'formatted.xml' (field id 'i' in 'EnhancedConfigurationData' section)
              // This is where you would dynamically set:
              // - Currency types
              // - Cassette types and denominations (related to notes_in_cassettes, notes_rejected, notes_dispensed)
              // - Timers (for screen display, input timeouts, etc.)
              // - FDK masks (if defined here)
              // - Other hardware-specific configurations
              default:
                this.log.warn(`  Unhandled Enhanced Config Parameter ID: ${param.id} with value: ${param.value}`);
                break;
            }
          });
          return this.replySolicitedStatus('Ready');
        } else {
          this.log.error('ATM.processCustomizationCommand(): No parameters in Enhanced Configuration Data.');
          return this.replySolicitedStatus('Command Reject');
        }

      default:
        this.log.error('ATM.processCustomizationCommand(): unknown message identifier: ', data.message_identifier);
        return this.replySolicitedStatus('Command Reject');
    }
  }

  /**
   * Processes an Interactive Transaction Response from the host.
   * @param {object} data - Parsed Interactive Transaction Response message object.
   * @returns {object} A structured response message object.
   */
  processInteractiveTransactionResponse(data){
    this.interactive_transaction = true;

    if(data.active_keys){
      this.setFDKsActiveMask(data.active_keys);
    }
    
    // Assuming screen_data_field contains raw data for dynamic screen
    this.display.setScreen(this.screens.parseDynamicScreenData(data.screen_data_field));
    return this.replySolicitedStatus('Ready');
  }

  /**
   * Processes Extended Encryption Key Information from the host.
   * @param {object} data - Parsed Extended Encryption Key Information message object.
   * @returns {object} A structured response message object.
   */
  processExtendedEncKeyInfo(data){
    switch(data.modifier){
      case 'Decipher new comms key with current master key':
        // data.new_key_data is assumed to be raw decimal string, length in hex
        if (this.crypto.setCommsKey(data.new_key_data, data.new_key_length)) {
          return this.replySolicitedStatus('Ready');
        } else {
          return this.replySolicitedStatus('Command Reject');
        }

      default:
        this.log.error('ATM.processExtendedEncKeyInfo(): Unsupported modifier: ' + data.modifier);
        break;
    }
    return this.replySolicitedStatus('Command Reject');
  }

  /**
   * Dispatches incoming Data Command messages to their specific handlers.
   * @param {object} data - Parsed Data Command message object.
   * @returns {object} A structured response message object.
   */
  processDataCommand(data){
    switch(data.message_subclass){
      case 'Customization Command':
        return this.processCustomizationCommand(data);

      case 'Interactive Transaction Response':
        return this.processInteractiveTransactionResponse(data);

      case 'Extended Encryption Key Information':
        return this.processExtendedEncKeyInfo(data);
          
      default:
        this.log.warn('ATM.processDataCommand(): unknown message subclass: ', data.message_subclass);
        return this.replySolicitedStatus('Command Reject');
    }
  }

  /**
   * Processes an incoming Transaction Reply message from the host.
   * @param {object} data - Parsed Transaction Reply message object.
   * @returns {object} A structured response message object.
   */
  processTransactionReply(data){    
    // `data.next_state` will be directly available as a property from the parser's output.
    this.processState(data.next_state);

    if(data.screen_display_update)
      this.screens.parseScreenDisplayUpdate(data.screen_display_update); // screen_display_update might also need dynamic parsing

    // Here, you could also process notes_to_dispense, coins_to_dispense, printer_data etc.
    // by interacting with ATM hardware services to simulate dispensing/printing.
    if (data.notes_to_dispense && data.notes_to_dispense.length > 0) {
        this.log.info(`ATM: Dispensing notes: ${data.notes_to_dispense.join(', ')}`);
        // Update supply counters dynamically based on notes dispensed
        this.supply_counters.notes_dispensed = (parseInt(this.supply_counters.notes_dispensed, 10) + data.notes_to_dispense.length).toString().padStart(20, '0');
        // This is a simplified update, actual NDC tracks per denomination.
    }
    if (data.printer_data && data.printer_data.length > 0) {
        this.log.info(`ATM: Printing data:`, data.printer_data);
        // Simulate printing to journal/receipt
    }

    return this.replySolicitedStatus('Ready');
  }

  /**
   * Generates a Message Co-Ordination Number based on settings.
   * The MCN cycles through ASCII characters, used to link requests to replies.
   * @returns {string} The next Message Co-Ordination Number.
   */
  getMessageCoordinationNumber(){
    let saved = this.settings.get('message_coordination_number');
    if(!saved || saved.charCodeAt(0) < 0x31 || saved.charCodeAt(0) > 0x7E) // Default to '1' if invalid or not set
      saved = String.fromCharCode(0x30); // Start from '0' (ASCII 48) so that first increment makes it '1' (ASCII 49)

    let nextChar = String.fromCharCode(saved.charCodeAt(0) + 1);
    // If the next character goes beyond '7E' (or '3F' if MCN Range is 000), loop back to '1' (ASCII 31)
    if (nextChar.charCodeAt(0) > 0x7E) { // Assuming 0x31 to 0x7E range from doc
        nextChar = String.fromCharCode(0x31);
    }
    this.settings.set('message_coordination_number', nextChar);
    return nextChar;
  }

  /**
   * Initializes or clears the terminal buffers (PIN, General Purpose, Amount, Opcode, FDK).
   */
  initBuffers(){
    this.PIN_buffer = ''; // In this app, holds clear PIN. For encrypted PIN, use getEncryptedPIN()
    this.buffer_B = '';
    this.buffer_C = '';
    this.amount_buffer = '000000000000'; // Zero-filled, 12 digits
    this.opcode.init(); // Resets opcode buffer
    this.FDK_buffer = ''; // FDK_buffer is only needed on state type Y and W to determine the next state
    return true;
  }

  /**
   * Processes Card Read state (State A).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number, or null.
   */
  processStateA(state){
    this.initBuffers();
    this.display.setScreenByNumber(state.get('screen_number'));
    
    if(this.card) {
      this.log.info(`ATM: Card detected in State A. Proceeding to good read next state: ${state.get('good_read_next_state')}`);
      return state.get('good_read_next_state');
    }
    this.log.info("ATM: Waiting for card in State A.");
    return null; // Stay in current state until card is read
  }

  /**
   * Processes PIN Entry state (State B).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number, or null.
   */
  processPINEntryState(state){
    this.display.setScreenByNumber(state.get('screen_number'));
    // setFDKsActiveMask('001') -> '001' (decimal) corresponds to FDK 'A' being active.
    // This is hardcoded; if FDK mask comes from Enhanced Config, use that.
    this.setFDKsActiveMask('001'); // Enabling button 'A' only (Enter/Clear on keypad also apply)

    // PMXPN (Maximum PIN Digits Entered) comes from FIT entry.
    const maxPinLength = this.FITs.getMaxPINLength(this.card.number) || 6; // Default to 6 if FIT not found
    this.log.info(`ATM: PIN Entry. Max PIN length: ${maxPinLength}. Current PIN length: ${this.PIN_buffer.length}`);

    if (this.PIN_buffer.length >= maxPinLength || (this.PIN_buffer.length >= 4 && this.buttons_pressed.includes('enter'))) {
      const nextState = state.get('remote_pin_check_next_state');
      this.log.info(`ATM: PIN entered. Proceeding to PIN check state: ${nextState}`);
      this.buttons_pressed = []; // Clear buttons after action
      return nextState;
    }
    this.log.info("ATM: Waiting for PIN input in State B.");
    return null; // Stay in current state until enough PIN digits or Enter is pressed
  }

  /**
   * Processes Amount Entry state (State F).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number, or null.
   */
  processAmountEntryState(state){
    this.display.setScreenByNumber(state.get('screen_number'));
    // '015' (decimal) for FDKs A, B, C, D active (1+2+4+8=15)
    // This is hardcoded; if FDK mask comes from Enhanced Config, use that.
    this.setFDKsActiveMask('015');
    
    // Only initialize amount buffer if it's currently zero (new entry)
    if (this.amount_buffer === '000000000000') {
      this.amount_buffer = '000000000000';
    }

    const button = this.buttons_pressed.shift(); // Get the first button pressed
    if(button && this.isFDKButtonActive(button)){
      const nextState = state.get('FDK_' + button + '_next_state');
      this.log.info(`ATM: FDK ${button} pressed in Amount Entry. Next state: ${nextState}`);
      return nextState;
    }
    this.log.info("ATM: Waiting for amount input or FDK in State F.");
    return null; // Stay in current state
  }

  /**
   * Processes State D (Set Operation Code Buffer from State).
   * @param {object} state - The current state object.
   * @param {object} [extension_state] - Optional extension state object.
   * @returns {string} The next state number.
   */
  processStateD(state, extension_state){
    this.opcode.setBufferFromState(state, extension_state);
    const nextState = state.get('next_state');
    this.log.info(`ATM: Processed State D. Next state: ${nextState}`);
    return nextState;
  }

  /**
   * Processes Four FDK Selection State (State E).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number, or null.
   */
  processFourFDKSelectionState(state){
    this.display.setScreenByNumber(state.get('screen_number'));

    // Dynamically activate FDKs based on whether their next_state is not '255' (no exit)
    this.activeFDKs = [];
    ['A', 'B', 'C', 'D'].forEach((element) => {
      if(state.get('FDK_' + element + '_next_state') !== '255') {
        this.activeFDKs.push(element);
      }
    });

    const button = this.buttons_pressed.shift();
    if(button && this.isFDKButtonActive(button)){
      let index = parseInt(state.get('buffer_location'), 10);
      if(index < 8) { // Operation Code buffer has 8 positions (0-7)
        this.opcode.setBufferValueAt(7 - index, button); // Assuming 7-index for reverse order
      } else {
        this.log.error('Invalid buffer location value: ' + state.get('buffer_location') + '. Operation Code buffer is not changed');
      }
      const nextState = state.get('FDK_' + button + '_next_state');
      this.log.info(`ATM: FDK ${button} pressed in State E. Next state: ${nextState}`);
      return nextState;
    }
    this.log.info("ATM: Waiting for FDK input in State E.");
    return null; // Stay in current state
  }

  processInformationEntryState(state){
    this.display.setScreenByNumber(state.get('screen_number'));
    
    // Build active mask dynamically based on next states for FDKs A,B,C,D
    let active_mask_binary_str = '0'; // Numeric Keys activator bit placeholder
    ['A', 'B', 'C', 'D'].forEach( element => {
      if(state.get('FDK_' + element + '_next_state') !== '255')
        active_mask_binary_str += '1';
      else
        active_mask_binary_str += '0';
    });
    this.setFDKsActiveMask(active_mask_binary_str); // Set using binary mask string

    const button = this.buttons_pressed.shift();
    if(button && this.isFDKButtonActive(button)){
      const nextState = state.get('FDK_' + button + '_next_state');
      this.log.info(`ATM: FDK ${button} pressed in Information Entry State. Next state: ${nextState}`);
      return nextState;
    }

    // Clear buffers based on buffer_and_display_params (index 2 for display parameter)
    const display_param = state.get('buffer_and_display_params')[2];
    switch(display_param){
      case '0': // Store data in general-purpose Buffer C, display 'X'
      case '1': // Store data in general-purpose Buffer C, display as keyed
        this.buffer_C = '';
        break;
      case '2': // Store data in general-purpose Buffer B, display 'X'
      case '3': // Store data in general-purpose Buffer B, display as keyed
        this.buffer_B = '';
        break;
      default: 
        this.log.error('Unsupported Display parameter value for Information Entry State: ' + display_param);
    }
    this.log.info("ATM: Waiting for input or FDK in Information Entry State.");
    return null; // Stay in current state
  }

  processInteractiveTransaction(request){
    this.interactive_transaction = false; // Reset flag after processing

    // Keyboard data entered after receiving an Interactive Transaction Response is stored in General Purpose Buffer B
    const button = this.buttons_pressed.shift();
    if (button) {
      this.buffer_B = button; // Store the button (e.g., numeric input, FDK pressed)
      request.buffer_B = button; // Add to transaction request for host
      this.log.info(`ATM: Interactive Transaction processed. Buffer B updated with: ${button}`);
    }
    return request;
  }

  /**
   * Processes Transaction Request State (State I).
   * Builds the Transaction Request message data to be sent to the host.
   * @param {object} state - The current state object.
   * @returns {object|null} The structured Transaction Request object, or null.
   */
  processTransactionRequestState(state){
    this.display.setScreenByNumber(state.get('screen_number'));

    // Construct the structured Transaction Request object
    let requestData = {
      messageId: 'TransactionRequest', // Maps to "TransactionRequest" in XML definitions
      data: {
        luno: this.settings.get('host').luno || '009', // ATM's LUNO
        top_of_receipt: '1', // Default as per original code
        message_coordination_number: this.getMessageCoordinationNumber(),
        time_variant_number: new Date().toISOString().replace(/[^0-9]/g, '').substring(0, 8), // Dynamic timestamp for Time Variant Number
      }
    };

    if (!this.interactive_transaction) {
      // Logic for non-interactive transactions (most common flow)
      if(state.get('send_track2') === '001') {
        requestData.data.track2 = this.track2;
        this.log.info(`ATM: Adding Track 2 to Transaction Request: ${this.track2}`);
      }

      // Track 1 and Track 3 options (if supported by your NDC version and needed)
      // if(state.get('send_track1') === '001') { /* add requestData.data.track1 = this.track1; */ }
      // if(state.get('send_track3') === '001') { /* add requestData.data.track3 = this.track3; */ }

      if(state.get('send_operation_code') === '001') {
        requestData.data.opcode_buffer = this.opcode.getBuffer();
        this.log.info(`ATM: Adding Opcode Buffer: ${requestData.data.opcode_buffer}`);
      }

      if(state.get('send_amount_data') === '001') {
        requestData.data.amount_buffer = this.amount_buffer;
        this.log.info(`ATM: Adding Amount Buffer: ${requestData.data.amount_buffer}`);
      }

      switch(state.get('send_pin_buffer')){
        case '001': // Standard format. Send Buffer A (encrypted PIN)
        case '129': // Extended format. Send Buffer A (encrypted PIN)
          if (this.PIN_buffer && this.card && this.card.number) {
            requestData.data.PIN_buffer = this.crypto.getEncryptedPIN(this.PIN_buffer, this.card.number);
            this.log.info(`ATM: Adding Encrypted PIN Buffer.`);
          } else {
            this.log.warn("ATM: PIN buffer, card, or card number missing for PIN encryption.");
          }
          break;
        case '000': // Standard format. Do not send Buffer A
        case '128': // Extended format. Do not send Buffer A
        default:
          this.log.info("ATM: PIN buffer not configured to be sent.");
          break;
      }

      switch(state.get('send_buffer_B_buffer_C')){
        case '000': /* Send no buffers */ break;
        case '001': requestData.data.buffer_B = this.buffer_B; break;
        case '002': requestData.data.buffer_C = this.buffer_C; break;
        case '003': 
          requestData.data.buffer_B = this.buffer_B;
          requestData.data.buffer_C = this.buffer_C;
          break;
        default:
          // TODO: If the extended format is selected in table entry 8, this entry is an Extension state number.
          // This typically means the next state handles complex buffer sending.
          if(state.get('send_pin_buffer') === '128' || state.get('send_pin_buffer') === '129'){
            this.log.warn("ATM: Extended buffer sending for B/C not fully implemented, relying on extension state logic.");
          } else {
            this.log.error(`ATM: Unsupported send_buffer_B_buffer_C value: ${state.get('send_buffer_B_buffer_C')}`);
          }
          break;
      }
    } else {
      // Interactive transaction flow
      requestData = this.processInteractiveTransaction(requestData);
      this.log.info("ATM: Processing interactive transaction request.");
    }

    this.transaction_request = requestData; // Store the structured request object for sending
    this.log.info("ATM: Transaction request prepared:", requestData);
    return null; // Stay in current state; listener will pick up transaction_request
  }

  /**
   * Processes Close State (State J).
   * @param {object} state - The current state object.
   * @returns {null} Always returns null as it's a terminal state.
   */
  processCloseState(state){
    this.display.setScreenByNumber(state.get('receipt_delivered_screen'));
    this.setFDKsActiveMask('000');  // Disable all FDK buttons
    this.card = null;
    this.log.info('ATM: Close State processed. Receipt screen shown.');
    // In a real ATM, this would trigger cash dispensing, card return/capture etc.
    return null; // This state typically doesn't transition immediately
  }

  /**
   * Processes State K (Financial Institution Identification).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number based on FITs, or null if no match.
   */
  processStateK(state){
    let institution_id = this.FITs.getInstitutionByCardnumber(this.card.number);
    this.log.info('ATM: Card ' + this.card.number + ' matches with institution_id ' + institution_id);
    if(institution_id) {
      const nextState = state.get('state_exits')[parseInt(institution_id, 10)];
      this.log.info(`ATM: Matched FIT. Proceeding to state: ${nextState}`);
      return nextState;
    } else {
      this.log.error('ATM: Unable to get Financial Institution by card number.');
      // Fallback or error state handling if no FIT match
      // You might want to define a default next state for no match
      return null;
    }
  }

  /**
   * Processes State W (Look-up Table Selection from FDK).
   * @param {object} state - The current state object.
   * @returns {string|null} The next state number.
   */
  processStateW(state){
    const nextState = state.get('states')[this.FDK_buffer];
    if (nextState) {
        this.log.info(`ATM: FDK Buffer ${this.FDK_buffer} in State W maps to next state: ${nextState}`);
        return nextState;
    }
    this.log.warn(`ATM: FDK Buffer ${this.FDK_buffer} in State W has no defined next state.`);
    return null;
  }

  /**
   * Assigns the provided value to the amount buffer.
   * @param {string} amount - The numeric amount string.
   */
  setAmountBuffer(amount){
    if (!amount) return;
    // Pad to 12 digits, keeping only last 12 chars if more are entered.
    this.amount_buffer = (this.amount_buffer.length + amount.length > 12)
                          ? this.amount_buffer.substring(amount.length) + amount
                          : this.amount_buffer.padStart(12 - amount.length, '0') + amount;
    this.log.info(`ATM: Amount buffer updated to: ${this.amount_buffer}`);
  }

  /**
   * Processes State X (Store data and activate FDKs).
   * @param {object} state - The current state object.
   * @param {object} [extension_state] - Optional extension state object.
   * @returns {string|null} The next state number, or null.
   */
  processStateX(state, extension_state){
    this.display.setScreenByNumber(state.get('screen_number'));
    this.setFDKsActiveMask(state.get('FDK_active_mask'));

    const button = this.buttons_pressed.shift();
    if(button && this.isFDKButtonActive(button)){
      this.FDK_buffer = button; // Store FDK pressed

      if(extension_state){
        let buffer_value_from_ext_state;
        // Map FDK button to entry in extension state
        const FDK_TO_EXT_INDEX = { 'A': 2, 'B': 3, 'C': 4, 'D': 5, 'F': 6, 'G': 7, 'H': 8, 'I': 9 };
        const extIndex = FDK_TO_EXT_INDEX[button];
        if (extIndex !== undefined) {
          buffer_value_from_ext_state = extension_state.get('entries')[extIndex];
        } else {
          this.log.warn(`ATM: FDK ${button} has no mapped entry in extension state for State X.`);
        }

        /**
         * Buffer ID identifies which buffer is to be edited and the number of zeros to add
         * to the values specified in the Extension state:
         * 01X - General purpose buffer B
         * 02X - General purpose buffer C
         * 03X - Amount buffer
         * X specifies the number of zeros in the range 0-9
         */
        if (buffer_value_from_ext_state !== undefined) {
          const buffer_id = state.get('buffer_id');
          const num_of_zeroes = parseInt(buffer_id.substring(2, 3), 10); // X specifies number of zeros

          let final_buffer_value = buffer_value_from_ext_state;
          for (let i = 0; i < num_of_zeroes; i++) {
            final_buffer_value += '0';
          }

          switch(buffer_id.substring(1, 2)){ // Check which buffer to use (1 for B, 2 for C, 3 for Amount)
            case '1': this.buffer_B = final_buffer_value; break;
            case '2': this.buffer_C = final_buffer_value; break;
            case '3': this.setAmountBuffer(final_buffer_value); break;
            default: this.log.error('Unsupported buffer id value in State X: ' + buffer_id); break;
          }
          this.log.info(`ATM: State X processed. Buffer ${buffer_id.substring(1, 2)} updated with value from extension state.`);
        }
      }

      const nextState = state.get('FDK_next_state');
      this.log.info(`ATM: FDK ${button} pressed in State X. Next state: ${nextState}`);
      return nextState;
    }
    this.log.info("ATM: Waiting for FDK input in State X.");
    return null; // Stay in current state
  }

  /**
   * Processes State Y (Store FDK to Opcode Buffer).
   * @param {object} state - The current state object.
   * @param {object} [extension_state] - Optional extension state object.
   * @returns {string|null} The next state number, or null.
   */
  processStateY(state, extension_state){
    this.display.setScreenByNumber(state.get('screen_number'));
    this.setFDKsActiveMask(state.get('FDK_active_mask'));

    if(extension_state){
      this.log.error('ATM: Extension state on state Y is not yet supported. Implement logic based on your spec.');
      // You would implement logic here to use the extension state's entries for buffer values
    } else {
      const button = this.buttons_pressed.shift();
      if(button && this.isFDKButtonActive(button)){
        this.FDK_buffer = button;
        // If no extension state, state.get('buffer_positions') defines Opcode buffer position (000-007).
        this.opcode.setBufferValueAt(parseInt(state.get('buffer_positions'), 10), button);
        const nextState = state.get('FDK_next_state');
        this.log.info(`ATM: FDK ${button} pressed in State Y. Opcode buffer position ${state.get('buffer_positions')} updated. Next state: ${nextState}`);
        return nextState;
      }
    }
    this.log.info("ATM: Waiting for FDK input in State Y.");
    return null; // Stay in current state
  }

  /**
   * Processes State Begin ICC Init (State +).
   * @param {object} state - The current state object.
   * @returns {string} The next state number.
   */
  processStateBeginICCInit(state){
    this.log.info(`ATM: Processing State Begin ICC Init. Next state: ${state.get('icc_init_not_started_next_state')}`);
    return state.get('icc_init_not_started_next_state');
  }

  /**
   * Processes State Complete ICC Application Initialization (State /).
   * @param {object} state - The current state object.
   * @returns {string} The next state number.
   */
  processStateCompleteICCAppInit(state){
    let extension_state = this.states.get(state.get('extension_state'));
    this.display.setScreenByNumber(state.get('please_wait_screen_number'));
    this.log.info(`ATM: Processing State Complete ICC App Init. Next state: ${extension_state.get('entries')[8]} (Processing not performed)`);
    return extension_state.get('entries')[8]; // Processing not performed
  }

  /**
   * Processes ICC Re-initialization State (State ;).
   * @param {object} state - The current state object.
   * @returns {string} The next state number.
   */
  processICCReinit(state){
    this.log.info(`ATM: Processing ICC Reinit. Next state: ${state.get('processing_not_performed_next_state')}`);
    return state.get('processing_not_performed_next_state');
  }

  /**
   * Processes Set ICC Data State (State ?).
   * @param {object} state - The current state object.
   * @returns {string} The next state number.
   */
  processSetICCDataState(state){
    // No processing as ICC cards are not currently supported in this simulator logic
    this.log.info(`ATM: Processing Set ICC Data State. Next state: ${state.get('next_state')} (No ICC processing).`);
    return state.get('next_state');
  }

  /**
   * Main state machine processor. Transitions ATM through states based on input/events.
   * @param {string} state_number - The number of the state to process.
   * @returns {boolean} True if state processing completes, false if an error occurs.
   */
  processState(state_number){
    let state = this.states.get(state_number);
    let next_state_number = null;

    let processingLoopCount = 0; // Guard against infinite loops
    const MAX_STATE_LOOP_ITERATIONS = 20; // Prevent runaway state transitions

    do {
      if (processingLoopCount++ > MAX_STATE_LOOP_ITERATIONS) {
          this.log.error(`ATM.processState(): Exceeded max state loop iterations (${MAX_STATE_LOOP_ITERATIONS}). Aborting state processing.`);
          return false;
      }

      if(state){
        this.current_state = state;
        this.log.info(`ATM: Processing state ${state.get('number')}${state.get('type')} (${state.get('description')})`);
      } else {
        this.log.error(`ATM: Error getting state ${state_number}: state not found`);
        return false;
      }
        
      switch(state.get('type')){
        case 'A': next_state_number = this.processStateA(state); break;
        case 'B': next_state_number = this.processPINEntryState(state); break;
        case 'D':
            state.get('extension_state') !== '255' && state.get('extension_state') !== '000'
                ? next_state_number = this.processStateD(state, this.states.get(state.get('extension_state')))
                : next_state_number = this.processStateD(state);
            break;
        case 'E': next_state_number = this.processFourFDKSelectionState(state); break;
        case 'F': next_state_number = this.processAmountEntryState(state); break;
        case 'H': next_state_number = this.processInformationEntryState(state); break;
        case 'I': next_state_number = this.processTransactionRequestState(state); break;
        case 'J': next_state_number = this.processCloseState(state); break;
        case 'K': next_state_number = this.processStateK(state); break;
        case 'X':
            (state.get('extension_state') !== '255' && state.get('extension_state') !== '000')
                ? next_state_number = this.processStateX(state, this.states.get(state.get('extension_state')))
                : next_state_number = this.processStateX(state);
            break;
        case 'Y':
            (state.get('extension_state') !== '255' && state.get('extension_state') !== '000')
                ? next_state_number = this.processStateY(state, this.states.get(state.get('extension_state')))
                : next_state_number = this.processStateY(state);
            break;
        case 'W': next_state_number = this.processStateW(state); break;
        case '+': next_state_number = this.processStateBeginICCInit(state); break;
        case '/': next_state_number = this.processStateCompleteICCAppInit(state); break;
        case ';': next_state_number = this.processICCReinit(state); break;
        case '?': next_state_number = this.processSetICCDataState(state); break;
        default:
          this.log.error('ATM.processState(): unsupported state type ' + state.get('type'));
          next_state_number = null; // No transition
          break;
      }

      if (next_state_number) {
        state_number = next_state_number; // Update for next iteration
        state = this.states.get(state_number); // Get the next state object
        if (!state) { // Handle case where next state number is invalid
            this.log.error(`ATM: Next state ${state_number} not found.`);
            return false;
        }
        // If a button was pressed to trigger the state, clear it.
        // If the state transition was internal, keep buttons for next possible state
        if (this.buttons_pressed.length > 0 && next_state_number === this.current_state.get('number')) {
             // If stayed in same state, and a button triggered, keep button
        } else {
             // For transitions, clear buttons.
             this.buttons_pressed = [];
        }
      } else {
        // No explicit next state means the ATM stays in the current state or waits for input.
        // Exit the loop.
        break;
      }
    } while (next_state_number); // Continue as long as a state transition occurs

    return true;
  }

  /**
   * Parses track2 data from a card and returns a structured card object.
   * @param {string} track2_data - The raw track2 string.
   * @returns {object|null} A structured card object, or null if parsing fails.
   */
  parseTrack2(track2_data){
    let card = {};
    try{
      let splitted = track2_data.split('=');
      card.track2 = track2_data;
      card.number = splitted[0].replace(';', ''); // Remove leading ';'
      card.service_code = splitted[1].substring(4, 7); // Assuming service code is at pos 4, length 3
      // You might also extract expiry date, PVV, etc. if they are consistently in track2_data
      // Based on CardsService, expiry_date and other fields are separate.
      // For full parsing, you'd need the exact track2 format.
    }catch(e){
      this.log.error('ATM.parseTrack2(): Error parsing track2 data: ' + e.message);
      return null;
    }
    return card;
  }

  /**
   * Processes a card read event.
   * @param {string} cardnumber - The card number.
   * @param {string} track2_data_portion - The track2 data portion (after card number and '=')
   */
  readCard(cardnumber, track2_data_portion){
    // Construct the full Track 2 data as it would appear on the magnetic stripe
    this.track2 = `;${cardnumber}=${track2_data_portion}`; // Standard Track 2 format starts with ';'
    this.card = this.parseTrack2(this.track2); // Re-parse full track2
    
    if(this.card){
      this.log.info('ATM: Card ' + this.card.number + ' read successfully. Track2: ' + this.track2);
      this.setStatus('Processing Card'); // Change status immediately
      // The `processState('000')` will transition to Card Read State (A) and then proceed based on card.
      this.processState(this.hostConfig.initial_screen_number || '000'); // Go to initial state after card read
    } else {
      this.log.error('ATM: Failed to parse card data.');
      this.setStatus('Out-Of-Service'); // Or a specific error state
    }
  }

  /**
   * Initializes or resets supply counters.
   * These can be overridden by Enhanced Configuration Data from the host.
   */
  initCounters(){
    let config_id_from_settings = this.settings.get('config_id');
    this.setConfigID(config_id_from_settings || '0000'); // Use saved config ID or default

    this.supply_counters = {
      tsn: '0000', // Transaction Sequence Number
      transaction_count: '0000000',
      notes_in_cassettes: '00011000220003300044', // Example static. Can be dynamic from config.
      notes_rejected: '00000000000000000000', // Example static.
      notes_dispensed: '00000000000000000000', // Example static.
      last_trxn_notes_dispensed: '00000000000000000000', // Example static.
      card_captured: '00000',
      envelopes_deposited: '00000',
      camera_film_remaining: '00000',
      last_envelope_serial: '00000',
    };
    this.log.info('ATM: Initialized supply counters.');
    // If you receive specific counter values from Enhanced Config Data, update them here
    // e.g., in processCustomizationCommand -> Enhanced Configuration Data Load
  }

  /**
   * Retrieves current supply counters.
   * @returns {object} The current supply counters.
   */
  getSupplyCounters(){
    return this.supply_counters;
  }

  /**
   * Sets the ATM's configuration ID and saves it to settings.
   * @param {string} config_id - The configuration ID.
   */
  setConfigID(config_id){
    this.config_id = config_id;
    this.settings.set('config_id', config_id);
    this.log.info(`ATM: Config ID set to: ${this.config_id}`);
  }

  /**
   * Retrieves the ATM's current configuration ID.
   * @returns {string} The configuration ID.
   */
  getConfigID(){
    return this.config_id;
  }

  /**
   * Sets the ATM's operational status and updates the display accordingly.
   * @param {string} status - The new status ('Offline', 'Connected', 'In-Service', 'Out-Of-Service', 'Processing Card').
   */
  setStatus(status){
    this.status = status;
    this.log.info(`ATM Status Changed to: ${status}`);

    // If status implies a default screen, set it.
    switch(status){
      case 'Offline':
      case 'Out-Of-Service':
        this.display.setScreenByNumber(this.hostConfig.initial_screen_number || '001'); // Use dynamic initial screen
        break;
      // You might have other status-driven screen changes here
    }
  }

  /**
   * Processes an incoming host message (which is already parsed).
   * This is the central dispatcher for all messages from the host.
   * @param {object} data - The structured and parsed message object received from the host.
   * @returns {object|null} A structured response message object to be sent back to the host, or null.
   */
  processHostMessage(data){
    this.log.info(`ATM: Processing incoming host message. Message ID: ${data.message_id || 'Unknown'}, Class: ${data.message_class}`);
    switch(data.message_class){
      case 'Terminal Command':
        return this.processTerminalCommand(data);

      case 'Data Command':
        return this.processDataCommand(data);

      case 'Transaction Reply Command':
        return this.processTransactionReply(data);
            
      case 'EMV Configuration':
        // EMV configuration messages typically load data into EMV tables.
        // For now, just acknowledge. Future work involves processing data.
        this.log.info(`ATM: Received EMV Configuration message (${data.message_subclass}).`);
        return this.replySolicitedStatus('Ready');

      default:
        this.log.warn('ATM.processHostMessage(): Unhandled message class: ' + data.message_class);
        return this.replySolicitedStatus('Command Reject'); // Reject unknown messages
    }
  }

  /**
   * Processes an FDK (Function Display Key) button press.
   * @param {string} button - The FDK button pressed (e.g., 'A', 'B').
   */
  processFDKButtonPressed(button){
    this.log.info(`ATM: FDK ${button} button pressed.`);
    // The behavior of FDKs depends heavily on the current state.
    switch(this.current_state.get('type')){
      case 'B': // PIN Entry State
        if (button.toUpperCase() === 'A' && this.PIN_buffer.length >= 4) {
          // If FDK 'A' (or Enter) is pressed in PIN entry, and PIN is at least 4 digits
          this.buttons_pressed.push('enter'); // Simulate enter press for PIN length check
          this.processState(this.current_state.get('number'));
        } else {
          this.log.warn(`ATM: FDK ${button} ignored in PIN Entry State B.`);
        }
        break;

      case 'H': // Information Entry State
        // Recalculate active mask as state might clear it or previous logic might have.
        let active_mask_h = '0';
        [ this.current_state.get('FDK_A_next_state'),
          this.current_state.get('FDK_B_next_state'),
          this.current_state.get('FDK_C_next_state'),
          this.current_state.get('FDK_D_next_state')].forEach( element => {
          if(element !== '255')
            active_mask_h += '1';
          else
            active_mask_h += '0';
        });
        this.setFDKsActiveMask(active_mask_h);

        if(this.isFDKButtonActive(button)){
          this.buttons_pressed.push(button);
          this.processState(this.current_state.get('number'));
        } else {
          this.log.warn(`ATM: FDK ${button} is not active in Information Entry State H.`);
        }
        break;

      default:
        // For other states, just push the button and re-process current state.
        this.buttons_pressed.push(button);
        this.processState(this.current_state.get('number'));
        break;
    }
  }

  /**
   * Processes a Backspace button press from the pinpad.
   */
  processBackspaceButtonPressed(){
    switch(this.current_state.get('type')){
      case 'B': // PIN Entry State
        this.PIN_buffer = this.PIN_buffer.slice(0, -1);
        this.display.insertText(this.PIN_buffer, '*'); // Display masked PIN
        this.log.info("ATM: Backspace in PIN Entry. Current PIN buffer length: " + this.PIN_buffer.length);
        break;
      case 'F': // Amount Entry State
        this.amount_buffer = '0' + this.amount_buffer.substring(0, this.amount_buffer.length - 1);
        this.display.insertText(this.amount_buffer);
        this.log.info("ATM: Backspace in Amount Entry. Current amount buffer: " + this.amount_buffer);
        break;
      case 'H': // Information Entry State
        {
          let bufferToModify;
          let maskingChar;
          const display_param = this.current_state.get('buffer_and_display_params')[2];

          switch(display_param){
            case '0': // Buffer C, display 'X'
              this.buffer_C = this.buffer_C.slice(0, -1);
              bufferToModify = this.buffer_C;
              maskingChar = 'X';
              break;
            case '1': // Buffer C, display as keyed
              this.buffer_C = this.buffer_C.slice(0, -1);
              bufferToModify = this.buffer_C;
              break;
            case '2': // Buffer B, display 'X'
              this.buffer_B = this.buffer_B.slice(0, -1);
              bufferToModify = this.buffer_B;
              maskingChar = 'X';
              break;
            case '3': // Buffer B, display as keyed
              this.buffer_B = this.buffer_B.slice(0, -1);
              bufferToModify = this.buffer_B;
              break;
            default:
              this.log.error('ATM: Unsupported Display parameter value for Backspace in H State: ' + display_param);
              return;
          }
          this.display.insertText(bufferToModify, maskingChar);
          this.log.info(`ATM: Backspace in Information Entry H. Buffer ${display_param < 2 ? 'C' : 'B'} updated.`);
        }
        break;
      default:
        this.log.warn(`ATM: Backspace ignored in state type ${this.current_state.get('type')}.`);
        break;
    }
  }

  /**
   * Processes an Enter button press from the pinpad.
   */
  processEnterButtonPressed(){
    switch(this.current_state.get('type')){
      case 'B': // PIN Entry State
        // If PIN length is sufficient (at least 4 as per standard NDC)
        if(this.PIN_buffer.length >= 4) {
          this.buttons_pressed.push('enter'); // Add 'enter' to signal state machine
          this.processState(this.current_state.get('number')); // Re-process current state to trigger transition
        } else {
          this.log.warn("ATM: Enter pressed in PIN Entry, but PIN is too short (min 4 digits).");
          this.display.insertText(this.PIN_buffer, '*'); // Update display just in case
        }
        break;
      case 'F': // Amount Entry State
        // If the cardholder presses Enter, it has the same effect as pressing FDK 'A' (often 'OK')
        this.buttons_pressed.push('A'); // Simulate FDK 'A' press
        this.processState(this.current_state.get('number'));
        this.log.info("ATM: Enter pressed in Amount Entry, simulating FDK A press.");
        break;
      default:
        this.log.warn(`ATM: Enter key ignored in state type ${this.current_state.get('type')}.`);
        break;
    }
  }

  /**
   * Processes an Escape button press from the pinpad.
   */
  processEscButtonPressed(){
    switch(this.current_state.get('type')){
      case 'B': // PIN Entry State
        this.PIN_buffer = ''; // Clear PIN buffer
        this.display.insertText(this.PIN_buffer, '*'); // Clear display
        this.log.info("ATM: Escape pressed in PIN Entry. PIN buffer cleared.");
        // Often leads to a 'Cancel' state or previous screen, but not explicit here.
        // You might want to trigger a specific state transition here if 'esc' has a global effect.
        break;
      default:
        this.log.warn(`ATM: Escape key ignored in state type ${this.current_state.get('type')}.`);
        break;
    }
  }

  /**
   * Processes a numeric button press from the pinpad.
   * @param {string} button - The numeric digit pressed.
   */
  processNumericButtonPressed(button){
    switch(this.current_state.get('type')){
      case 'B': // PIN Entry State
        this.PIN_buffer += button;
        const maxPinLength = this.FITs.getMaxPINLength(this.card.number) || 6;
        this.display.insertText(this.PIN_buffer, '*'); // Display masked PIN
        this.log.info(`ATM: Numeric ${button} pressed in PIN Entry. Current PIN length: ${this.PIN_buffer.length}`);
        // If PIN length reaches max, automatically trigger state transition (as if Enter was pressed)
        if(this.PIN_buffer.length >= maxPinLength) {
          this.buttons_pressed.push('enter'); // Simulate enter press
          this.processState(this.current_state.get('number'));
        }
        break;
      case 'F': // Amount Entry State
        // Shift left and add new digit to the right
        this.amount_buffer = this.amount_buffer.substring(1) + button;
        this.display.insertText(this.amount_buffer);
        this.log.info(`ATM: Numeric ${button} pressed in Amount Entry. Current amount: ${this.amount_buffer}`);
        break;
      case 'H': // Information Entry State
        {
          let keyDisplayChar;
          let bufferToUpdate;
          const display_param = this.current_state.get('buffer_and_display_params')[2];

          switch(display_param){
            case '0': // Buffer C, display 'X'
            case '1': // Buffer C, display as keyed
              if(this.buffer_C.length < 32){ // Max length 32
                this.buffer_C += button;
                bufferToUpdate = this.buffer_C;
                keyDisplayChar = (display_param === '0') ? 'X' : undefined;
              }
              break;
            case '2': // Buffer B, display 'X'
            case '3': // Buffer B, display as keyed
              if(this.buffer_B.length < 32){ // Max length 32
                this.buffer_B += button;
                bufferToUpdate = this.buffer_B;
                keyDisplayChar = (display_param === '2') ? 'X' : undefined;
              }
              break;
            default:
              this.log.error('ATM: Unsupported Display parameter value for Numeric press in H State: ' + display_param);
              break;
          }
          if(bufferToUpdate) {
            this.display.insertText(bufferToUpdate, keyDisplayChar);
            this.log.info(`ATM: Numeric ${button} pressed in Information Entry H. Buffer ${display_param < 2 ? 'C' : 'B'} updated.`);
          }
        }
        break;
      default:
        this.log.warn(`ATM: No keyboard entry allowed for state type ${this.current_state.get('type')}.`);
        break;
    }
  }

  /**
   * Dispatches pinpad button presses to their specific handlers.
   * @param {string} button - The button pressed ('backspace', 'enter', 'esc', or a digit '0'-'9').
   */
  processPinpadButtonPressed(button){
    switch(button){
      case 'backspace': this.processBackspaceButtonPressed(); break;
      case 'enter': this.processEnterButtonPressed(); break;
      case 'esc': this.processEscButtonPressed(); break;
      default: this.processNumericButtonPressed(button); break;
    }
  }

  /**
   * Retrieves the current state object.
   * @returns {object} The current state object.
   */
  getCurrentState(){
    return this.current_state;
  }
}

module.exports = ATM;
