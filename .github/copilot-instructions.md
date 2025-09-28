# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### GSM-SMS Adapter Specific Context

This is the **ioBroker.gsmsms** adapter, designed for sending and receiving SMS messages using GSM hardware connected via serial port. Key characteristics:

- **Primary Function:** Communication with GSM modems through serial port to send/receive SMS
- **Hardware Integration:** Supports various GSM modules and USB dongles with AT command interface
- **Message Formats:** Supports both PDU mode (recommended) and text mode for SMS handling
- **Serial Communication:** Uses serialport-gsm library for GSM modem communication
- **AT Commands:** Implements custom AT command execution for modem control
- **Multi-language Support:** Handles international SMS with proper encoding
- **Configuration Areas:**
  - Serial port settings (baudrate, parity, stop bits)
  - GSM specific settings (PIN, signal monitoring, CNMI settings)
  - Message handling preferences (auto-delete, concatenation)

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      adapter = new AdapterInstance();
    });
    
    afterEach(() => {
      adapter.unload();
    });
    
    it('should initialize correctly', () => {
      expect(adapter).toBeDefined();
    });
  });
  ```

### GSM-SMS Specific Testing Patterns

When working with the gsmsms adapter, use these testing patterns:

- **Mock Serial Port Communication:** Use sinon or jest mocks for serialport-gsm
- **AT Command Testing:** Create mock responses for common AT commands (AT+CMGF, AT+CNMI, etc.)
- **PDU Message Parsing:** Test SMS PDU encoding/decoding with sample messages
- **Error Scenarios:** Test modem disconnection, invalid PIN, signal loss
- **Example mock structure:**
  ```javascript
  const mockGSM = {
    initializeModem: jest.fn(),
    sendMessage: jest.fn(),
    deleteMessage: jest.fn(),
    executeCommand: jest.fn()
  };
  ```

### Integration Testing
- Test the complete adapter lifecycle (startup, configuration, messaging, shutdown)
- Use `@iobroker/testing` framework for comprehensive adapter testing
- Test state changes and object creation
- Verify proper cleanup in unload method

## Code Style & Architecture

### General ioBroker Adapter Patterns
- Use `@iobroker/adapter-core` as the base class
- Implement proper lifecycle methods: `onReady()`, `onUnload()`, `onStateChange()`
- Use appropriate logging levels: `error`, `warn`, `info`, `debug`
- Handle uncaught exceptions gracefully
- Implement proper resource cleanup in `unload()` method

### GSM-SMS Specific Patterns

- **Serial Port Management:** Always close connections in unload()
- **AT Command Structure:** Use consistent command formatting and error handling
- **Message Queuing:** Implement proper queue management for outgoing SMS
- **State Management:** Update connection status and signal quality states
- **Example patterns:**
  ```javascript
  // Proper AT command execution
  async executeATCommand(command, expectResponse = true) {
    try {
      const response = await this.modem.executeCommand(command);
      this.log.debug(`AT Command: ${command} -> ${response}`);
      return response;
    } catch (error) {
      this.log.error(`AT Command failed: ${command} - ${error.message}`);
      throw error;
    }
  }
  
  // SMS sending with proper error handling
  async sendSMS(recipient, message, alert = false) {
    if (!this.modemConnected) {
      throw new Error('Modem not connected');
    }
    // Implementation here
  }
  ```

### Error Handling
- Wrap external API calls in try-catch blocks
- Use specific error types and messages
- Log errors appropriately based on severity
- Implement retry mechanisms for transient failures
- Always clean up resources on errors

### Configuration Handling
- Use `io-package.json` native section for configuration schema
- Validate configuration values before use
- Provide sensible defaults
- Use encrypted storage for sensitive data (PIN codes)

## Logging Best Practices

### Standard Logging Levels
- **Error:** Critical failures that prevent adapter operation
- **Warn:** Important issues that don't stop operation but need attention
- **Info:** General operational information and state changes  
- **Debug:** Detailed diagnostic information for troubleshooting

### GSM-SMS Specific Logging
- Log AT commands at debug level with responses
- Log connection status changes at info level
- Log SMS send/receive events at info level
- Log signal quality changes at debug level
- Example logging:
  ```javascript
  this.log.info(`SMS sent to ${recipient}: ${message.length} chars`);
  this.log.debug(`Modem response: ${response}`);
  this.log.warn(`Low signal quality: ${signalQuality}%`);
  this.log.error(`Failed to initialize modem: ${error.message}`);
  ```

## State Management

### ioBroker State Patterns
- Create states with proper roles and types
- Use `ack: true` for device states, `ack: false` for commands
- Implement proper state change handlers
- Clean up custom states on adapter restart

### GSM-SMS State Structure
The adapter uses these state categories:
- **admin.*** - Configuration and AT command states
- **info.*** - Read-only device information (connection, signal, etc.)
- **sendSMS.*** - Outgoing message handling
- **inbox.*** - Incoming message information

Example state creation:
```javascript
await this.setObjectNotExistsAsync('info.connection', {
  type: 'state',
  common: {
    name: 'Modem connected',
    type: 'boolean',
    role: 'indicator.connected',
    read: true,
    write: false,
    def: false
  },
  native: {}
});
```

## Async/Promise Handling

### Modern JavaScript Patterns
- Use `async/await` instead of callbacks where possible
- Handle promise rejections properly
- Implement timeout mechanisms for external calls
- Use Promise.all() for parallel operations when appropriate

### GSM Communication Patterns
- Implement timeouts for AT commands (typically 5-30 seconds)
- Use queuing for sequential command execution
- Handle modem not responding scenarios
- Example async pattern:
  ```javascript
  async initializeModem() {
    try {
      await this.executeCommand('ATZ'); // Reset
      await this.delay(1000);
      await this.executeCommand('ATE0'); // Disable echo
      await this.executeCommand(`AT+CPIN=${this.config.pin}`); // Enter PIN
    } catch (error) {
      this.log.error(`Modem initialization failed: ${error.message}`);
      throw error;
    }
  }
  ```

## Package.json and Dependencies

### Required Dependencies
- `@iobroker/adapter-core`: Core adapter functionality
- Main functionality dependencies based on adapter purpose

### GSM-SMS Dependencies
- `serialport-gsm`: GSM modem communication library
- Handle version compatibility and security updates

### Development Dependencies
- `@iobroker/testing`: Testing framework
- `@iobroker/eslint-config`: Code style enforcement
- `mocha`, `chai`: Unit testing
- TypeScript definitions for proper type checking

## Documentation

### README Requirements
- Clear installation instructions
- Configuration parameter explanations  
- Supported hardware list
- Troubleshooting section
- Changelog with version history

### GSM-SMS Documentation Specifics
- List of tested GSM modems/dongles
- Serial port configuration examples
- AT command reference for advanced users
- SMS encoding and special characters handling
- PIN security considerations

### Code Comments
- Document complex algorithms and business logic
- Explain hardware-specific implementations
- Use JSDoc for function documentation
- Comment AT command sequences and their purposes

## Security Considerations

### ioBroker Security Standards
- Never log sensitive configuration data
- Use encrypted native fields for passwords/tokens
- Validate all user inputs
- Follow principle of least privilege

### GSM-SMS Security
- Encrypt SIM PIN in configuration (using `encryptedNative`)
- Validate phone numbers format
- Implement rate limiting for SMS sending
- Log security events appropriately
- Example encrypted config:
  ```javascript
  // In io-package.json native section
  "encryptedNative": ["pin"]
  ```

## Hardware Integration Patterns

### Serial Port Communication
- Always check port availability before connecting
- Implement proper connection timeouts
- Handle port disconnection gracefully
- Use appropriate baud rates and serial settings

### GSM Modem Communication
- Initialize with proper AT command sequence
- Handle different modem response formats
- Implement signal quality monitoring
- Support various SMS storage modes
- Example initialization sequence:
  ```javascript
  const initCommands = [
    'ATZ',              // Reset modem
    'ATE0',             // Disable command echo
    'AT+CMGF=0',        // PDU mode
    'AT+CNMI=2,1,0,2,0' // New message indications
  ];
  ```

## Performance Optimization

### Efficient Resource Usage
- Minimize memory allocations in loops
- Use streams for large data processing
- Implement proper connection pooling
- Cache frequently accessed data

### GSM-SMS Performance
- Use message queuing to prevent overloading modem
- Implement batch operations where possible
- Cache AT command responses when appropriate
- Monitor and optimize serial port buffer sizes

### Monitoring and Metrics
- Track key performance indicators
- Log performance metrics at debug level
- Implement health checks
- Monitor resource usage over time

## Platform Compatibility

### Node.js Versions
- Support current LTS versions
- Test compatibility across supported versions
- Use appropriate polyfills if needed
- Follow ioBroker minimum requirements

### Operating System Support
- Ensure cross-platform compatibility (Windows, Linux, macOS)
- Handle platform-specific serial port differences
- Test on different hardware architectures
- Document platform-specific requirements

### GSM Hardware Compatibility
- Support common USB-to-serial GSM modems
- Handle different AT command dialects
- Test with various cellular network configurations
- Document known compatible hardware

## Debugging and Diagnostics

### Debugging Tools
- Use adapter log levels effectively for debugging
- Implement diagnostic commands and states
- Provide clear error messages with context
- Create helper tools for common debugging tasks

### GSM-SMS Diagnostics
- Implement AT command testing interface
- Provide signal quality monitoring
- Log detailed modem communication at debug level
- Create diagnostic states for troubleshooting
- Example diagnostic implementation:
  ```javascript
  // AT command testing state
  this.subscribeStates('admin.atCommand*');
  
  onStateChange(id, state) {
    if (id.includes('atCommand') && state && !state.ack) {
      this.executeATCommand(state.val)
        .then(response => {
          this.setState('admin.atResponse', response, true);
        })
        .catch(error => {
          this.setState('admin.atResponse', `ERROR: ${error.message}`, true);
        });
    }
  }
  ```

## Version Control and Release Management

### Git Workflow
- Use meaningful commit messages
- Follow semantic versioning
- Tag releases appropriately
- Maintain clean commit history

### Release Process
- Use `@alcalzone/release-script` for automated releases
- Update changelog with each release
- Test thoroughly before releases
- Follow ioBroker release guidelines

### ioBroker Repository Integration
- Ensure compatibility with ioBroker repository requirements
- Keep io-package.json updated
- Follow adapter certification process
- Maintain backwards compatibility when possible

## Code Examples and Snippets

### Common ioBroker Patterns

#### Adapter Initialization
```javascript
class GsmSms extends utils.Adapter {
  constructor(options = {}) {
    super({
      ...options,
      name: 'gsmsms',
    });
    
    this.on('ready', this.onReady.bind(this));
    this.on('stateChange', this.onStateChange.bind(this));
    this.on('unload', this.onUnload.bind(this));
  }
  
  async onReady() {
    // Initialize adapter
    await this.initializeModem();
    await this.subscribeStates('*');
  }
}
```

#### State Management
```javascript
// Create and set states
await this.setObjectNotExistsAsync('info.connection', {
  type: 'state',
  common: {
    name: 'Connection Status',
    type: 'boolean',
    role: 'indicator.connected',
    read: true,
    write: false,
    def: false
  },
  native: {}
});

// Update state with acknowledgment
await this.setStateAsync('info.connection', true, true);
```

### GSM-SMS Specific Examples

#### SMS Sending
```javascript
async sendSMS(recipient, message, alertClass = false) {
  try {
    if (!this.modemConnected) {
      throw new Error('Modem not connected');
    }
    
    const messageObj = {
      recipient: recipient,
      message: message,
      alert: alertClass
    };
    
    const response = await this.modem.sendMessage(messageObj, {
      request: 'sendSMS'
    });
    
    this.log.info(`SMS sent to ${recipient}: "${message}" (${message.length} chars)`);
    await this.updateOutboxState(messageObj);
    
    return response;
  } catch (error) {
    this.log.error(`Failed to send SMS to ${recipient}: ${error.message}`);
    throw error;
  }
}
```

#### AT Command Execution
```javascript
async executeATCommand(command, timeout = 5000) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      reject(new Error(`AT command timeout: ${command}`));
    }, timeout);
    
    this.modem.executeCommand(command, (response, error) => {
      clearTimeout(timer);
      if (error) {
        this.log.debug(`AT Command failed: ${command} -> ${error}`);
        reject(new Error(error));
      } else {
        this.log.debug(`AT Command: ${command} -> ${response}`);
        resolve(response);
      }
    });
  });
}
```

#### Modem Initialization
```javascript
async initializeModem() {
  try {
    this.log.info('Initializing GSM modem...');
    
    // Initialize modem with settings
    const modemOptions = {
      baudRate: this.config.baudRate || 19200,
      dataBits: this.config.dataBits || 8,
      stopBits: this.config.stopBits || 1,
      parity: this.config.parity || 'none',
      rtscts: this.config.rtscts || false,
      xon: this.config.xon || false,
      xoff: this.config.xoff || false,
      xany: this.config.xany || false,
      autoInitOnOpen: true,
      enableConcatenation: this.config.enableConcatenation,
      incomingCallIndication: this.config.incomingCallIndication,
      incomingSMSIndication: this.config.incomingSMSIndication,
      pin: this.config.pin || '',
      customInitCommand: this.config.customInitCommand || '',
      logger: this.log
    };
    
    this.modem = new SerialPort(this.config.port, modemOptions);
    
    // Setup event handlers
    this.modem.on('onNewMessage', this.onNewMessage.bind(this));
    this.modem.on('onError', this.onModemError.bind(this));
    
    this.log.info('GSM modem initialized successfully');
    await this.setStateAsync('info.connection', true, true);
    
  } catch (error) {
    this.log.error(`Failed to initialize modem: ${error.message}`);
    await this.setStateAsync('info.connection', false, true);
    throw error;
  }
}
```

#### Message Handling
```javascript
onNewMessage(message) {
  try {
    this.log.info(`New SMS from ${message.sender}: "${message.text}"`);
    
    // Update inbox states
    this.setStateAsync('inbox.messageText', message.text, true);
    this.setStateAsync('inbox.messageSender', message.sender, true);
    this.setStateAsync('inbox.messageDate', message.dateTime, true);
    this.setStateAsync('inbox.messageRaw', JSON.stringify(message), true);
    
    // Auto-delete if configured
    if (this.config.autoDeleteOnReceive) {
      this.modem.deleteMessage(message.index);
    }
    
  } catch (error) {
    this.log.error(`Error processing new message: ${error.message}`);
  }
}
```

## Additional Resources

- [ioBroker Adapter Development Documentation](https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterdev.md)
- [ioBroker Best Practices](https://github.com/ioBroker/ioBroker.repositories#development-and-coding-best-practices)
- [serialport-gsm Documentation](https://www.npmjs.com/package/serialport-gsm)
- [AT Commands Reference](https://www.developershome.com/sms/atCommandsIntro.asp)