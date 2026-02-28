# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.5.6
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

---

## 📑 Table of Contents

1. [Project Context](#project-context)
2. [Code Quality & Standards](#code-quality--standards)
   - [Code Style Guidelines](#code-style-guidelines)
   - [ESLint Configuration](#eslint-configuration)
3. [Testing](#testing)
   - [Unit Testing](#unit-testing)
   - [Integration Testing](#integration-testing)
   - [API Testing with Credentials](#api-testing-with-credentials)
4. [Development Best Practices](#development-best-practices)
   - [Dependency Management](#dependency-management)
   - [HTTP Client Libraries](#http-client-libraries)
   - [Error Handling](#error-handling)
5. [Admin UI Configuration](#admin-ui-configuration)
   - [JSON-Config Setup](#json-config-setup)
   - [Translation Management](#translation-management)
6. [Documentation](#documentation)
   - [README Updates](#readme-updates)
   - [Changelog Management](#changelog-management)
7. [CI/CD & GitHub Actions](#cicd--github-actions)
   - [Workflow Configuration](#workflow-configuration)
   - [Testing Integration](#testing-integration)
8. [GSM-SMS Adapter Specific Implementations](#gsm-sms-adapter-specific-implementations)

---

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

---

## Code Quality & Standards

### Code Style Guidelines

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

**Timer and Resource Cleanup Example:**
```javascript
private connectionTimer?: NodeJS.Timeout;

async onReady() {
  this.connectionTimer = setInterval(() => this.checkConnection(), 30000);
}

onUnload(callback) {
  try {
    if (this.connectionTimer) {
      clearInterval(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    callback();
  } catch (e) {
    callback();
  }
}
```

### ESLint Configuration

**CRITICAL:** ESLint validation must run FIRST in your CI/CD pipeline, before any other tests. This "lint-first" approach catches code quality issues early.

#### Setup
```bash
npm install --save-dev eslint @iobroker/eslint-config
```

#### Configuration (.eslintrc.json)
```json
{
  "extends": "@iobroker/eslint-config",
  "rules": {
    // Add project-specific rule overrides here if needed
  }
}
```

#### Package.json Scripts
```json
{
  "scripts": {
    "lint": "eslint --max-warnings 0 .",
    "lint:fix": "eslint . --fix"
  }
}
```

#### Best Practices
1. ✅ Run ESLint before committing — fix ALL warnings, not just errors
2. ✅ Use `lint:fix` for auto-fixable issues
3. ✅ Don't disable rules without documentation
4. ✅ Lint all relevant files (main code, tests, build scripts)
5. ✅ Keep `@iobroker/eslint-config` up to date
6. ✅ **ESLint warnings are treated as errors in CI** (`--max-warnings 0`). The `lint` script above already includes this flag — run `npm run lint` to match CI behavior locally

#### Common Issues
- **Unused variables**: Remove or prefix with underscore (`_variable`)
- **Missing semicolons**: Run `npm run lint:fix`
- **Indentation**: Use 4 spaces (ioBroker standard)
- **console.log**: Replace with `adapter.log.debug()` or remove

---

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
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Get all states created by adapter
                        const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');
                        
                        console.log(`📊 Found ${stateIds.length} states`);

                        if (stateIds.length > 0) {
                            console.log('✅ Adapter successfully created states');
                            
                            // Show sample of created states
                            const allStates = await new Promise((res, rej) => {
                                harness.states.getStates(stateIds, (err, states) => {
                                    if (err) return rej(err);
                                    res(states || []);
                                });
                            });
                            
                            console.log('📋 Sample states created:');
                            stateIds.slice(0, 5).forEach((stateId, index) => {
                                const state = allStates[index];
                                console.log(`   ${stateId}: ${state && state.val !== undefined ? state.val : 'undefined'}`);
                            });
                            
                            await harness.stopAdapter();
                            resolve(true);
                        } else {
                            console.log('❌ No states were created by the adapter');
                            reject(new Error('Adapter did not create any states'));
                        }
                    } catch (error) {
                        reject(error);
                    }
                });
            }).timeout(40000);
        });
    }
});
```

#### Testing Both Success AND Failure Scenarios

**IMPORTANT**: For every "it works" test, implement corresponding "it doesn't work and fails" tests. This ensures proper error handling and validates that your adapter fails gracefully when expected.

```javascript
// Example: Testing successful configuration
it('should configure and start adapter with valid configuration', function () {
    return new Promise(async (resolve, reject) => {
        // ... successful configuration test as shown above
    });
}).timeout(40000);

// Example: Testing failure scenarios
it('should NOT create daily states when daily is disabled', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));
            console.log('✅ Step 1.5: Adapter object loaded');

            console.log('🔍 Step 2: Updating adapter config...');
            Object.assign(obj.native, {
                position: TEST_COORDINATES,
                createCurrently: false,
                createHourly: true,
                createDaily: false, // Daily disabled for this test
            });

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    console.log('✅ Step 2.5: Adapter object updated');
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();
            console.log('✅ Step 4: Adapter started');

            console.log('⏳ Step 5: Waiting 20 seconds for states...');
            await new Promise((res) => setTimeout(res, 20000));

            console.log('🔍 Step 6: Fetching state IDs...');
            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');

            console.log(`📊 Step 7: Found ${stateIds.length} total states`);

            const hourlyStates = stateIds.filter((key) => key.includes('hourly'));
            if (hourlyStates.length > 0) {
                console.log(`✅ Step 8: Correctly ${hourlyStates.length} hourly weather states created`);
            } else {
                console.log('❌ Step 8: No hourly states created (test failed)');
                return reject(new Error('Expected hourly states but found none'));
            }

            // Check daily states should NOT be present
            const dailyStates = stateIds.filter((key) => key.includes('daily'));
            if (dailyStates.length === 0) {
                console.log(`✅ Step 9: No daily states found as expected`);
            } else {
                console.log(`❌ Step 9: Daily states present (${dailyStates.length}) (test failed)`);
                return reject(new Error('Expected no daily states but found some'));
            }

            await harness.stopAdapter();
            console.log('🛑 Step 10: Adapter stopped');

            resolve(true);
        } catch (error) {
            reject(error);
        }
    });
}).timeout(40000);

// Example: Testing missing required configuration  
it('should handle missing required configuration properly', function () {
    return new Promise(async (resolve, reject) => {
        try {
            harness = getHarness();
            
            console.log('🔍 Step 1: Fetching adapter object...');
            const obj = await new Promise((res, rej) => {
                harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                    if (err) return rej(err);
                    res(o);
                });
            });
            
            if (!obj) return reject(new Error('Adapter object not found'));

            console.log('🔍 Step 2: Removing required configuration...');
            // Remove required configuration to test failure handling
            delete obj.native.position; // This should cause failure or graceful handling

            await new Promise((res, rej) => {
                harness.objects.setObject(obj._id, obj, (err) => {
                    if (err) return rej(err);
                    res(undefined);
                });
            });

            console.log('🔍 Step 3: Starting adapter...');
            await harness.startAdapterAndWait();

            console.log('⏳ Step 4: Waiting for adapter to process...');
            await new Promise((res) => setTimeout(res, 10000));

            console.log('🔍 Step 5: Checking adapter behavior...');
            const stateIds = await harness.dbConnection.getStateIDs('your-adapter.0.*');

            // Check if adapter handled missing configuration gracefully
            if (stateIds.length === 0) {
                console.log('✅ Adapter properly handled missing configuration - no invalid states created');
                resolve(true);
            } else {
                // If states were created, check if they're in error state
                const connectionState = await new Promise((res, rej) => {
                    harness.states.getState('your-adapter.0.info.connection', (err, state) => {
                        if (err) return rej(err);
                        res(state);
                    });
                });
                
                if (!connectionState || connectionState.val === false) {
                    console.log('✅ Adapter properly failed with missing configuration');
                    resolve(true);
                } else {
                    console.log('❌ Adapter should have failed or handled missing config gracefully');
                    reject(new Error('Adapter should have handled missing configuration'));
                }
            }

            await harness.stopAdapter();
        } catch (error) {
            console.log('✅ Adapter correctly threw error with missing configuration:', error.message);
            resolve(true);
        }
    });
}).timeout(40000);
```

#### Advanced State Access Patterns

For testing adapters that create multiple states, use bulk state access methods to efficiently verify large numbers of states:

```javascript
it('should create and verify multiple states', () => new Promise(async (resolve, reject) => {
    // Configure and start adapter first...
    harness.objects.getObject('system.adapter.tagesschau.0', async (err, obj) => {
        if (err) {
            console.error('Error getting adapter object:', err);
            reject(err);
            return;
        }

        // Configure adapter as needed
        obj.native.someConfig = 'test-value';
        harness.objects.setObject(obj._id, obj);

        await harness.startAdapterAndWait();

        // Wait for adapter to create states
        setTimeout(() => {
            // Access bulk states using pattern matching
            harness.dbConnection.getStateIDs('tagesschau.0.*').then(stateIds => {
                if (stateIds && stateIds.length > 0) {
                    harness.states.getStates(stateIds, (err, allStates) => {
                        if (err) {
                            console.error('❌ Error getting states:', err);
                            reject(err); // Properly fail the test instead of just resolving
                            return;
                        }

                        // Verify states were created and have expected values
                        const expectedStates = ['tagesschau.0.info.connection', 'tagesschau.0.articles.0.title'];
                        let foundStates = 0;
                        
                        for (const stateId of expectedStates) {
                            if (allStates[stateId]) {
                                foundStates++;
                                console.log(`✅ Found expected state: ${stateId}`);
                            } else {
                                console.log(`❌ Missing expected state: ${stateId}`);
                            }
                        }

                        if (foundStates === expectedStates.length) {
                            console.log('✅ All expected states were created successfully');
                            resolve();
                        } else {
                            reject(new Error(`Only ${foundStates}/${expectedStates.length} expected states were found`));
                        }
                    });
                } else {
                    reject(new Error('No states found matching pattern tagesschau.0.*'));
                }
            }).catch(reject);
        }, 20000); // Allow more time for multiple state creation
    });
})).timeout(45000);
```

#### Key Integration Testing Rules

1. **NEVER test API URLs directly** - Let the adapter handle API calls
2. **ALWAYS use the harness** - `getHarness()` provides the testing environment  
3. **Configure via objects** - Use `harness.objects.setObject()` to set adapter configuration
4. **Start properly** - Use `harness.startAdapterAndWait()` to start the adapter
5. **Check states** - Use `harness.states.getState()` to verify results
6. **Use timeouts** - Allow time for async operations with appropriate timeouts
7. **Test real workflow** - Initialize → Configure → Start → Verify States

#### Workflow Dependencies
Integration tests should run ONLY after lint and adapter tests pass:

```yaml
integration-tests:
  needs: [check-and-lint, adapter-tests]
  runs-on: ubuntu-latest
  steps:
    - name: Run integration tests
      run: npx mocha test/integration-*.js --exit
```

#### What NOT to Do
❌ Direct API testing: `axios.get('https://api.example.com')`
❌ Mock adapters: `new MockAdapter()`  
❌ Direct internet calls in tests
❌ Bypassing the harness system

#### What TO Do
✅ Use `@iobroker/testing` framework
✅ Configure via `harness.objects.setObject()`
✅ Start via `harness.startAdapterAndWait()`
✅ Test complete adapter lifecycle
✅ Verify states via `harness.states.getState()`
✅ Allow proper timeouts for async operations

### API Testing with Credentials
For adapters that connect to external APIs requiring authentication, implement comprehensive credential testing:

#### Password Encryption for Integration Tests
When creating integration tests that need encrypted passwords (like those marked as `encryptedNative` in io-package.json):

1. **Read system secret**: Use `harness.objects.getObjectAsync("system.config")` to get `obj.native.secret`
2. **Apply XOR encryption**: Implement the encryption algorithm:
   ```javascript
   async function encryptPassword(harness, password) {
       const systemConfig = await harness.objects.getObjectAsync("system.config");
       if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
           throw new Error("Could not retrieve system secret for password encryption");
       }
       
       const secret = systemConfig.native.secret;
       let result = '';
       for (let i = 0; i < password.length; ++i) {
           result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
       }
       return result;
   }
   ```
3. **Store encrypted password**: Set the encrypted result in adapter config, not the plain text
4. **Result**: Adapter will properly decrypt and use credentials, enabling full API connectivity testing

#### Demo Credentials Testing Pattern
- Use provider demo credentials when available (e.g., `demo@api-provider.com` / `demo`)
- Create separate test file (e.g., `test/integration-demo.js`) for credential-based tests
- Add npm script: `"test:integration-demo": "mocha test/integration-demo --exit"`
- Implement clear success/failure criteria with recognizable log messages
- Expected success pattern: Look for specific adapter initialization messages
- Test should fail clearly with actionable error messages for debugging

#### Enhanced Test Failure Handling
```javascript
it("Should connect to API with demo credentials", async () => {
    // ... setup and encryption logic ...
    
    const connectionState = await harness.states.getStateAsync("adapter.0.info.connection");
    
    if (connectionState && connectionState.val === true) {
        console.log("✅ SUCCESS: API connection established");
        return true;
    } else {
        throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
            "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
    }
}).timeout(120000); // Extended timeout for API calls
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

---

## Development Best Practices

### Dependency Management

- Always use `npm` for dependency management in ioBroker adapters
- Use `npm ci` for installing existing dependencies (respects package-lock.json)
- Use `npm install` only when adding or updating dependencies
- Keep dependencies minimal and focused
- Only update dependencies in separate Pull Requests

**When modifying package.json:**
1. Run `npm install` to sync package-lock.json
2. Commit both package.json and package-lock.json together

**Best Practices:**
- Prefer built-in Node.js modules when possible
- Use `@iobroker/adapter-core` for adapter base functionality
- Avoid deprecated packages
- Document specific version requirements

### HTTP Client Libraries

- **Preferred:** Use native `fetch` API (Node.js 20+ required)
- **Avoid:** `axios` unless specific features are required

**Example with fetch:**
```javascript
try {
  const response = await fetch('https://api.example.com/data');
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  const data = await response.json();
} catch (error) {
  this.log.error(`API request failed: ${error.message}`);
}
```

**Other Recommendations:**
- **Logging:** Use adapter built-in logging (`this.log.*`)
- **Scheduling:** Use adapter built-in timers and intervals
- **File operations:** Use Node.js `fs/promises`
- **Configuration:** Use adapter config system

### Error Handling

- Always catch and log errors appropriately
- Use adapter log levels (error, warn, info, debug)
- Provide meaningful, user-friendly error messages
- Handle network failures gracefully
- Implement retry mechanisms where appropriate
- Always clean up timers, intervals, and resources in `unload()` method

**Example:**
```javascript
try {
  await this.connectToDevice();
} catch (error) {
  this.log.error(`Failed to connect to device: ${error.message}`);
  this.setState('info.connection', false, true);
  // Implement retry logic if needed
}
```

---

## Admin UI Configuration

### JSON-Config Setup

Use JSON-Config format for modern ioBroker admin interfaces.

**Example Structure:**
```json
{
  "type": "panel",
  "items": {
    "host": {
      "type": "text",
      "label": "Host address",
      "help": "IP address or hostname of the device"
    }
  }
}
```

**Guidelines:**
- ✅ Use consistent naming conventions
- ✅ Provide sensible default values
- ✅ Include validation for required fields
- ✅ Add tooltips for complex options
- ✅ Ensure translations for all supported languages (minimum English and German)
- ✅ Write end-user friendly labels, avoid technical jargon

### Translation Management

**CRITICAL:** Translation files must stay synchronized with `admin/jsonConfig.json`. Orphaned keys or missing translations cause UI issues and PR review delays.

#### Overview
- **Location:** `admin/i18n/{lang}/translations.json` for 11 languages (de, en, es, fr, it, nl, pl, pt, ru, uk, zh-cn)
- **Source of truth:** `admin/jsonConfig.json` - all `label` and `help` properties must have translations
- **Command:** `npm run translate` - auto-generates translations but does NOT remove orphaned keys
- **Formatting:** English uses tabs, other languages use 4 spaces

#### Critical Rules
1. ✅ Keys must match exactly with jsonConfig.json
2. ✅ No orphaned keys in translation files
3. ✅ All translations must be in native language (no English fallbacks)
4. ✅ Keys must be sorted alphabetically

#### Workflow for Translation Updates

**When modifying admin/jsonConfig.json:**

1. Make your changes to labels/help texts
2. Run automatic translation: `npm run translate`
3. Create validation script (`scripts/validate-translations.js`):

```javascript
const fs = require('fs');
const path = require('path');
const jsonConfig = JSON.parse(fs.readFileSync('admin/jsonConfig.json', 'utf8'));

function extractTexts(obj, texts = new Set()) {
    if (typeof obj === 'object' && obj !== null) {
        if (obj.label) texts.add(obj.label);
        if (obj.help) texts.add(obj.help);
        for (const key in obj) {
            extractTexts(obj[key], texts);
        }
    }
    return texts;
}

const requiredTexts = extractTexts(jsonConfig);
const languages = ['de', 'en', 'es', 'fr', 'it', 'nl', 'pl', 'pt', 'ru', 'uk', 'zh-cn'];
let hasErrors = false;

languages.forEach(lang => {
    const translationPath = path.join('admin', 'i18n', lang, 'translations.json');
    const translations = JSON.parse(fs.readFileSync(translationPath, 'utf8'));
    const translationKeys = new Set(Object.keys(translations));
    
    const missing = Array.from(requiredTexts).filter(text => !translationKeys.has(text));
    const orphaned = Array.from(translationKeys).filter(key => !requiredTexts.has(key));
    
    console.log(`\n=== ${lang} ===`);
    if (missing.length > 0) {
        console.error('❌ Missing keys:', missing);
        hasErrors = true;
    }
    if (orphaned.length > 0) {
        console.error('❌ Orphaned keys (REMOVE THESE):', orphaned);
        hasErrors = true;
    }
    if (missing.length === 0 && orphaned.length === 0) {
        console.log('✅ All keys match!');
    }
});

process.exit(hasErrors ? 1 : 0);
```

4. Run validation: `node scripts/validate-translations.js`
5. Remove orphaned keys manually from all translation files
6. Add missing translations in native languages
7. Run: `npm run lint && npm run test`

#### Add Validation to package.json

```json
{
  "scripts": {
    "translate": "translate-adapter",
    "validate:translations": "node scripts/validate-translations.js",
    "pretest": "npm run lint && npm run validate:translations"
  }
}
```

#### Translation Checklist

Before committing changes to admin UI or translations:
1. ✅ Validation script shows "All keys match!" for all 11 languages
2. ✅ No orphaned keys in any translation file
3. ✅ All translations in native language
4. ✅ Keys alphabetically sorted
5. ✅ `npm run lint` passes
6. ✅ `npm run test` passes
7. ✅ Admin UI displays correctly

---

## Documentation

### README Updates

#### Required Sections
1. **Installation** - Clear npm/ioBroker admin installation steps
2. **Configuration** - Detailed configuration options with examples
3. **Usage** - Practical examples and use cases
4. **Changelog** - Version history (use "## **WORK IN PROGRESS**" for ongoing changes)
5. **License** - License information (typically MIT for ioBroker adapters)
6. **Support** - Links to issues, discussions, community support

#### Documentation Standards
- Use clear, concise language
- Include code examples for configuration
- Add screenshots for admin interface when applicable
- Maintain multilingual support (minimum English and German)
- Always reference issues in commits and PRs (e.g., "fixes #xx")

#### Mandatory README Updates for PRs

For **every PR or new feature**, always add a user-friendly entry to README.md:

- Add entries under `## **WORK IN PROGRESS**` section
- Use format: `* (author) **TYPE**: Description of user-visible change`
- Types: **NEW** (features), **FIXED** (bugs), **ENHANCED** (improvements), **TESTING** (test additions), **CI/CD** (automation)
- Focus on user impact, not technical details

**Example:**
```markdown
## **WORK IN PROGRESS**

* (DutchmanNL) **FIXED**: Adapter now properly validates login credentials (fixes #25)
* (DutchmanNL) **NEW**: Added device discovery to simplify initial setup
```

### Changelog Management

Follow the [AlCalzone release-script](https://github.com/AlCalzone/release-script) standard.

#### Format Requirements

```markdown
# Changelog

<!--
  Placeholder for the next version (at the beginning of the line):
  ## **WORK IN PROGRESS**
-->

## **WORK IN PROGRESS**

- (author) **NEW**: Added new feature X
- (author) **FIXED**: Fixed bug Y (fixes #25)

## v0.1.0 (2023-01-01)
Initial release
```

#### Workflow Process
- **During Development:** All changes go under `## **WORK IN PROGRESS**`
- **For Every PR:** Add user-facing changes to WORK IN PROGRESS section
- **Before Merge:** Version number and date added when merging to main
- **Release Process:** Release-script automatically converts placeholder to actual version

#### Change Entry Format
- Format: `- (author) **TYPE**: User-friendly description`
- Types: **NEW**, **FIXED**, **ENHANCED**
- Focus on user impact, not technical implementation
- Reference issues: "fixes #XX" or "solves #XX"

---

## CI/CD & GitHub Actions

### Workflow Configuration

#### GitHub Actions Best Practices

**Must use ioBroker official testing actions:**
- `ioBroker/testing-action-check@v1` for lint and package validation
- `ioBroker/testing-action-adapter@v1` for adapter tests
- `ioBroker/testing-action-deploy@v1` for automated releases with Trusted Publishing (OIDC)

**Configuration:**
- **Node.js versions:** Test on 20.x, 22.x, 24.x
- **Platform:** Use ubuntu-22.04
- **Automated releases:** Deploy to npm on version tags (requires NPM Trusted Publishing)
- **Monitoring:** Include Sentry release tracking for error monitoring

#### Critical: Lint-First Validation Workflow

**ALWAYS run ESLint checks BEFORE other tests.** Benefits:
- Catches code quality issues immediately
- Prevents wasting CI resources on tests that would fail due to linting errors
- Provides faster feedback to developers
- Enforces consistent code quality

**Workflow Dependency Configuration:**
```yaml
jobs:
  check-and-lint:
    # Runs ESLint and package validation
    # Uses: ioBroker/testing-action-check@v1
    
  adapter-tests:
    needs: [check-and-lint]  # Wait for linting to pass
    # Run adapter unit tests
    
  integration-tests:
    needs: [check-and-lint, adapter-tests]  # Wait for both
    # Run integration tests
```

**Key Points:**
- The `check-and-lint` job has NO dependencies - runs first
- ALL other test jobs MUST list `check-and-lint` in their `needs` array
- If linting fails, no other tests run, saving time
- Fix all ESLint errors before proceeding

### Testing Integration

#### API Testing in CI/CD

For adapters with external API dependencies:

```yaml
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

#### Testing Best Practices
- Run credential tests separately from main test suite
- Don't make credential tests required for deployment
- Provide clear failure messages for API issues
- Use appropriate timeouts for external calls (120+ seconds)

#### Package.json Integration
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

---

## GSM-SMS Adapter Specific Implementations

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

### GSM-SMS Dependencies
- `serialport-gsm`: GSM modem communication library
- Handle version compatibility and security updates

### GSM-SMS Documentation Specifics
- List of tested GSM modems/dongles
- Serial port configuration examples
- AT command reference for advanced users
- SMS encoding and special characters handling
- PIN security considerations

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

### GSM-SMS Performance
- Use message queuing to prevent overloading modem
- Implement batch operations where possible
- Cache AT command responses when appropriate
- Monitor and optimize serial port buffer sizes

### GSM Hardware Compatibility
- Support common USB-to-serial GSM modems
- Handle different AT command dialects
- Test with various cellular network configurations
- Document known compatible hardware

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

