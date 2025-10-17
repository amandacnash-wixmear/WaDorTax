<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TaxSnap WA - Sales Tax Compliance Tool</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/feather-icons"></script>
    <script src="https://cdn.jsdelivr.net/npm/feather-icons/dist/feather.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/vanta@latest/dist/vanta.globe.min.js"></script>
    <style>
        .vanta-bg {
            position: absolute;
            width: 100%;
            height: 100%;
            z-index: -1;
        }
        .glass-card {
            backdrop-filter: blur(16px) saturate(180%);
            -webkit-backdrop-filter: blur(16px) saturate(180%);
            background-color: rgba(255, 255, 255, 0.75);
            border-radius: 12px;
            border: 1px solid rgba(209, 213, 219, 0.3);
        }
    </style>
</head>
<body class="min-h-screen bg-gray-50">
    <div class="vanta-bg" id="vanta-bg"></div>
    
    <div class="container mx-auto px-4 py-12">
        <!-- Header -->
        <header class="mb-12 text-center">
            <div class="inline-block p-4 glass-card">
                <h1 class="text-4xl md:text-5xl font-bold text-gray-800 mb-2">TaxSnap WA</h1>
                <p class="text-xl text-gray-600">The Washington State Sales Tax Compliance Tool</p>
            </div>
            <div class="mt-6 glass-card inline-block p-4">
                <p class="text-gray-700">
                    <i data-feather="alert-triangle" class="inline mr-2"></i>
                    Upload your sales data and we'll handle the tax calculations automatically
                </p>
            </div>
        </header>

        <!-- Main Content -->
        <main class="glass-card p-8 max-w-4xl mx-auto shadow-xl">
            <!-- File Upload Section -->
            <section class="mb-12">
                <h2 class="text-2xl font-semibold text-gray-800 mb-4">Upload Sales Data</h2>
                <div class="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center cursor-pointer hover:bg-gray-50 transition duration-200">
                    <div id="drop-area">
                        <i data-feather="upload-cloud" class="w-12 h-12 mx-auto text-blue-500 mb-4"></i>
                        <p class="text-gray-600 mb-2">Drag & drop your CSV/TSV file here</p>
                        <p class="text-sm text-gray-500 mb-4">or</p>
                        <label for="fileInput" class="px-6 py-2 bg-blue-600 text-white rounded-md cursor-pointer hover:bg-blue-700 transition duration-200">
                            Browse Files
                        </label>
                        <input id="fileInput" type="file" class="hidden" accept=".csv,.tsv">
                    </div>
                    <p class="text-xs text-gray-500 mt-4">Required columns: Full Address, Gross Sales, Net Sales, Transaction ID</p>
                </div>
            </section>

            <!-- Data Preview Section -->
            <section class="mb-12">
                <h2 class="text-2xl font-semibold text-gray-800 mb-4">Data Preview</h2>
                <div class="overflow-x-auto">
                    <table class="min-w-full bg-white rounded-lg overflow-hidden">
                        <thead class="bg-blue-600 text-white">
                            <tr>
                                <th class="py-3 px-4 text-left">Transaction ID</th>
                                <th class="py-3 px-4 text-left">Address</th>
                                <th class="py-3 px-4 text-left">Status</th>
                                <th class="py-3 px-4 text-left">Location Code</th>
                                <th class="py-3 px-4 text-left">Tax Rate</th>
                                <th class="py-3 px-4 text-left">Tax Amount</th>
                            </tr>
                        </thead>
                        <tbody id="dataPreview" class="divide-y divide-gray-200">
                            <tr class="text-center text-gray-500 py-4">
                                <td colspan="6" class="py-8">No data uploaded yet</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </section>

            <!-- Calculation Section -->
            <section>
                <h2 class="text-2xl font-semibold text-gray-800 mb-4">Tax Summary</h2>
                <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <div class="bg-blue-50 p-4 rounded-lg border border-blue-200">
                        <h3 class="text-blue-800 font-medium mb-2">Valid Transactions</h3>
                        <p id="validCount" class="text-3xl font-bold text-blue-600">0</p>
                    </div>
                    <div class="bg-yellow-50 p-4 rounded-lg border border-yellow-200">
                        <h3 class="text-yellow-800 font-medium mb-2">Invalid Addresses</h3>
                        <p id="invalidCount" class="text-3xl font-bold text-yellow-600">0</p>
                    </div>
                    <div class="bg-green-50 p-4 rounded-lg border border-green-200">
                        <h3 class="text-green-800 font-medium mb-2">Total Tax Due</h3>
                        <p id="totalTax" class="text-3xl font-bold text-green-600">$0.00</p>
                    </div>
                </div>
            </section>
        </main>

        <!-- Footer -->
        <footer class="mt-12 text-center text-gray-600">
            <div class="glass-card inline-block p-4">
                <p class="flex items-center justify-center">
                    <i data-feather="info" class="mr-2"></i>
                    Results are simulated based on current WA DOR tax rates
                </p>
                <div class="mt-2 flex justify-center space-x-4">
                    <a href="#" class="text-blue-600 hover:underline">Export JSON</a>
                    <a href="#" class="text-blue-600 hover:underline">Download Report</a>
                    <a href="#" class="text-blue-600 hover:underline">WA DOR Website</a>
                </div>
            </div>
        </footer>
    </div>

    <script>
        // Initialize Vanta.js globe background
        VANTA.GLOBE({
            el: "#vanta-bg",
            mouseControls: true,
            touchControls: true,
            gyroControls: false,
            minHeight: 200.00,
            minWidth: 200.00,
            scale: 1.00,
            scaleMobile: 1.00,
            color: 0x3b82f6,
            backgroundColor: 0xf8fafc
        });

        // File upload handling
        document.getElementById('fileInput').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file) {
                handleFileUpload(file);
            }
        });

        // Drag and drop handling
        const dropArea = document.getElementById('drop-area');
        ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
            dropArea.addEventListener(eventName, preventDefaults, false);
        });

        function preventDefaults(e) {
            e.preventDefault();
            e.stopPropagation();
        }

        ['dragenter', 'dragover'].forEach(eventName => {
            dropArea.addEventListener(eventName, highlight, false);
        });

        ['dragleave', 'drop'].forEach(eventName => {
            dropArea.addEventListener(eventName, unhighlight, false);
        });

        function highlight() {
            dropArea.classList.add('bg-blue-50', 'border-blue-400');
        }

        function unhighlight() {
            dropArea.classList.remove('bg-blue-50', 'border-blue-400');
        }

        dropArea.addEventListener('drop', function(e) {
            const dt = e.dataTransfer;
            const file = dt.files[0];
            handleFileUpload(file);
        });

        // Handle file upload and simulate processing
        function handleFileUpload(file) {
            const reader = new FileReader();
            reader.onload = function(e) {
                // Simulate processing (in a real app, this would parse the CSV/TSV)
                setTimeout(() => {
                    simulateTaxCalculation();
                }, 1000);
            };
            reader.readAsText(file);
        }

        // Simulate tax calculation with mock data
        function simulateTaxCalculation() {
            // Mock data that simulates processed transactions
            const mockData = [
                {
FYITransaction_ID: "TXN001",
                    Input_Address: "400 Pine St, Seattle, WA 98101",
                    Address_Status: "VALID",
                    DOR_Location_Code: "1710",
                    Total_Tax_Rate: 0.1010,
                    Gross_Sales_Final: 120.00,
                    Net_Sales_Final: 108.92,
                    Calculated_Tax_Amount: 11.08
                },
                {
                    Transaction_ID: "TXN002",
                    Input_Address: "505 Union Ave SE, Olympia, WA 98501",
                    Address_Status: "VALIDn",
                    DOR_Location_Code: "2230",
                    Total_Tax_Rate: 0.0890,
                    Gross_Sales_Final: 75.50,
                    Net_Sales_Final: 68.96,
                    Calculated_Tax_Amount: 6.54
                },
                {
                    Transaction_ID: "TXN003",
                    Input_Address: "123 Invalid St, Nowhere, XX 00000",
                    Address_Status: "INVALID_GEOCODE",
                    DOR_Location_Code: null,
                    Total_Tax_Rate: null,
                    Gross_Sales_Final: 200.00,
                    Net_Sales_Final: 200.00,
                    Calculated_Tax_Amount: null
                }
            ];

            // Update the data preview table
            const previewTable = document.getElementById('dataPreview');
            previewTable.innerHTML = '';
            
            let validCount = 0;
            let invalidCount = 0;
            let totalTax = 0;

            mockData.forEach(transaction => {
                const row = document.createElement('tr');
                
                if (transaction.Address_Status === 'VALID') {
                    validCount++;
                    totalTax += transaction.Calculated_Tax_Amount;
                    row.classList.add('bg-green-50');
                } else {
                    invalidCount++;
                    row.classList.add('bg-red-50');
                }

                row.innerHTML = `
                    <td class="py-3 px-4">${transaction.Transaction_ID}</td>
                    <td class="py-3 px-4">${transaction.Input_Address}</td>
                    <td class="py-3 px-4">
                        <span class="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium ${
                            transaction.Address_Status === 'VALID' ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'
                        }">
                            ${transaction.Address_Status}
                        </span>
                    </td>
                    <td class="py-3 px-4">${transaction.DOR_Location_Code || 'N/A'}</td>
                    <td class="py-3 px-4">${transaction.Total_Tax_Rate ? (transaction.Total_Tax_Rate * 100).toFixed(2) + '%' : 'N/A'}</td>
                    <td class="py-3 px-4">${transaction.Calculated_Tax_Amount ? '$' + transaction.Calculated_Tax_Amount.toFixed(2) : 'N/A'}</td>
                `;

                previewTable.appendChild(row);
            });

            // Update summary cards
            document.getElementById('validCount').textContent = validCount;
            document.getElementById('invalidCount').textContent = invalidCount;
            document.getElementById('totalTax').textContent = '$' + totalTax.toFixed(2);
        }

        // Initialize feather icons
        feather.replace();
    </script>
</body>
</html>

