<template>
    <div class="qr-generator">
        <div class="form-group">
            <label for="amount">Amount (KHR):</label>
            <input type="number" id="amount" v-model="amount" placeholder="Enter amount">
        </div>

        <button @click="generateQRCode" class="generate-btn" :disabled="isLoading">
            {{ isLoading ? 'Generating...' : 'Generate QR Code' }}
        </button>

        <div v-if="errorMessage" class="error-message">
            {{ errorMessage }}
        </div>

        <div v-if="qrCodeData" class="qr-display">
            <h2>Scan QR Code</h2>
            <QRCodeVue3 :value="qrCodeData" :size="250" :level="'H'" :margin="2" :background="'white'"
                :foreground="'black'" />
            <p class="payment-details">Payment to: Borenn Morm ({{ accountId }})</p>
            <p class="payment-amount">Amount: {{ formattedAmount }} KHR</p>
            <p class="payment-info">Transaction ID: {{ currentBillNumber }}</p>
            <p class="qr-instruction">Scan this QR code with your Bakong app to make a payment</p>
            <button @click="resetForm" class="reset-btn">Generate New QR Code</button>

            <!-- Debug information (hidden in production) -->
            <div v-if="debugInfo" class="debug-info">
                <h4>Debug Info:</h4>
                <pre>{{ JSON.stringify(debugInfo, null, 2) }}</pre>
            </div>
        </div>
    </div>
</template>

<script>
import QRCodeVue3 from 'qrcode.vue';

// Import necessary components from bakong-khqr
const bakongKhqr = require('bakong-khqr');
const { khqrData } = require('bakong-khqr/src/constant');
const { IndividualInfo } = require('bakong-khqr/src/model/information');

export default {
    name: 'BakongQRGenerator',
    components: {
        QRCodeVue3
    },
    data() {
        return {
            amount: null,
            qrCodeData: '',
            accountId: 'borenn_morm1@aclb',
            token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJkYXRhIjp7ImlkIjoiZGIwYWZiNDJkNTdlNDgxZiJ9LCJpYXQiOjE3NDIyOTE5NzQsImV4cCI6MTc1MDA2Nzk3NH0.G38llMAPB7lvuH1KomRi5iPb-bOuEuw-PAC64ELsf0g',
            isLoading: false,
            errorMessage: '',
            generatedAmount: 0,
            currentBillNumber: '',
            debugInfo: null
        };
    },
    computed: {
        formattedAmount() {
            return Number(this.generatedAmount).toLocaleString();
        }
    },
    methods: {
        generateQRCode() {
            if (!this.amount || isNaN(parseFloat(this.amount)) || parseFloat(this.amount) <= 0) {
                this.errorMessage = 'Please enter a valid amount greater than 0';
                return;
            }

            this.errorMessage = '';
            this.isLoading = true;

            // Generate a unique bill number for tracking
            const billNumberRaw = Math.floor(Math.random() * 10000).toString().padStart(4, '0');
            const billNumber = `BILL${billNumberRaw}`; // Format as BILL0001 instead of using # symbol

            try {
                // Set optional data for QR code
                // Format amount properly - remove decimal places for KHR
                const formattedAmount = Math.round(parseFloat(this.amount));

                const optionalData = {
                    currency: khqrData.currency.khr, // Use the proper currency code from the library
                    amount: formattedAmount, // Use rounded amount for KHR (no decimal places)
                    billNumber: billNumber.replace('#', ''), // Remove # as it might cause issues
                    storeLabel: "Borenn Store", // Use a consistent store name
                    terminalLabel: "Web Payment", // Add terminal label
                    mobileNumber: "85587575857", // Contact number
                    expirationTimestamp: Date.now() + (10 * 60 * 1000), // Expires in 10 minutes
                    merchantCategoryCode: "5999" // Standard merchant category code
                };

                // Create individual info object with proper merchant information
                const merchantName = "Borenn Morm"; // Must be consistent with Bakong registration
                const location = "Phnom Penh"; // Keep location shorter

                const individualInfo = new IndividualInfo(
                    this.accountId, // Bakong account ID
                    khqrData.currency.khr, // Currency code
                    merchantName, // Merchant name
                    location, // Location
                    optionalData
                );

                // Generate QR code with a small delay for better UX
                setTimeout(() => {
                    try {
                        // Initialize the Bakong KHQR library
                        const khqr = new bakongKhqr.BakongKHQR();

                        // Generate the QR code data
                        const response = khqr.generateIndividual(individualInfo);

                        if (response && response.data && response.data.qr) {
                            // Ensure QR data is properly formatted
                            this.qrCodeData = response.data.qr.trim(); // Use the qr property which contains the QR code string
                            this.generatedAmount = formattedAmount; // Use the formatted amount
                            this.currentBillNumber = billNumber; // Store the generated bill number

                            console.log('QR Code generated successfully:', response);
                            console.log('QR Data:', response.data);
                            console.log('QR String for scanning:', this.qrCodeData);

                            // Store debug info with detailed payment parameters
                            this.debugInfo = {
                                responseType: typeof response,
                                responseKeys: Object.keys(response),
                                status: response.status,
                                dataType: typeof response.data,
                                dataObject: response.data,
                                qrData: response.data.qr,
                                qrMD5: response.data.md5,
                                inputParams: {
                                    accountId: this.accountId,
                                    amount: formattedAmount,
                                    currency: khqrData.currency.khr,
                                    billNumber: billNumber,
                                    merchantName: merchantName,
                                    location: location,
                                    storeLabel: optionalData.storeLabel,
                                    terminalLabel: optionalData.terminalLabel
                                },
                                parsedQR: this.parseQRContent(response.data.qr),
                                timestamp: new Date().toISOString()
                            };

                            this.$emit('qr-generated', {
                                data: response.data,
                                amount: this.amount,
                                billNumber: billNumber
                            });
                        } else {
                            throw new Error('Invalid response from QR code generator');
                        }
                    } catch (error) {
                        console.error('Error generating QR code:', error);
                        this.errorMessage = 'Error generating QR code: ' + (error.message || 'Unknown error');
                    } finally {
                        this.isLoading = false;
                    }
                }, 500); // Add a slight delay for better UX
            } catch (error) {
                console.error('Error generating QR code:', error);
                this.errorMessage = 'Error generating QR code: ' + error.message;
                this.isLoading = false;
            }
        },

        resetForm() {
            this.qrCodeData = '';
            this.amount = null;
            this.errorMessage = '';
            this.generatedAmount = 0;
            this.currentBillNumber = '';
            this.isLoading = false;
            this.debugInfo = null;
            this.$emit('reset');
        },

        // Helper method to parse QR content for debugging
        parseQRContent(qrString) {
            if (!qrString) return null;

            try {
                // This is a basic parser for EMV QR codes
                // The format is typically TLV (Tag-Length-Value)
                const result = {};
                let i = 0;

                while (i < qrString.length) {
                    // Get tag (2 characters)
                    const tag = qrString.substring(i, i + 2);
                    i += 2;

                    // Get length (2 characters)
                    const length = parseInt(qrString.substring(i, i + 2), 10);
                    i += 2;

                    // Get value based on length
                    const value = qrString.substring(i, i + length);
                    i += length;

                    // Store in result object
                    result[tag] = value;
                }

                return result;
            } catch (error) {
                console.error('Error parsing QR content:', error);
                return { error: 'Failed to parse QR content', qrString: qrString };
            }
        }
    }
}
</script>

<style scoped>
.qr-generator {
    background-color: #f8f9fa;
    border-radius: 8px;
    padding: 20px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.form-group {
    margin-bottom: 20px;
    text-align: left;
}

label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
}

input {
    width: 100%;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 16px;
}

.generate-btn {
    background-color: #42b983;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 12px 20px;
    font-size: 16px;
    cursor: pointer;
    transition: background-color 0.3s;
}

.generate-btn:hover {
    background-color: #3aa876;
}

.generate-btn:disabled {
    background-color: #95d5bc;
    cursor: not-allowed;
}

.qr-display {
    margin-top: 30px;
    padding-top: 20px;
    border-top: 1px solid #ddd;
}

h2 {
    color: #2c3e50;
    margin-bottom: 15px;
}

.error-message {
    color: #e74c3c;
    margin-top: 10px;
    font-weight: 500;
}

.payment-details,
.payment-amount {
    margin-top: 10px;
    font-size: 16px;
    font-weight: 500;
}

.payment-amount {
    color: #2980b9;
    font-size: 18px;
    margin-bottom: 5px;
}

.payment-info {
    font-size: 14px;
    color: #34495e;
    margin-bottom: 10px;
}

.qr-instruction {
    font-style: italic;
    color: #7f8c8d;
    margin-bottom: 20px;
    font-size: 14px;
}

.reset-btn {
    background-color: #7f8c8d;
    color: white;
    border: none;
    border-radius: 4px;
    padding: 10px 15px;
    font-size: 14px;
    cursor: pointer;
    margin-top: 15px;
    transition: background-color 0.3s;
}

.reset-btn:hover {
    background-color: #6c7a7d;
}

.debug-info {
    margin-top: 20px;
    padding: 15px;
    background-color: #f5f5f5;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-family: monospace;
    font-size: 12px;
    text-align: left;
    overflow-x: auto;
}

.debug-info h4 {
    margin-top: 0;
    margin-bottom: 8px;
    color: #333;
}

.debug-info pre {
    margin: 0;
    white-space: pre-wrap;
}
</style>
