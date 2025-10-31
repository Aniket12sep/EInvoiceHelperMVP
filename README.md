# EInvoiceHelperMVP
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>5-Minute GST E-Invoice JSON Generator</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; background-color: #f4f7f6; }
        .container { background: #fff; padding: 30px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); }
        h2 { color: #007bff; text-align: center; }
        p.subtitle { text-align: center; color: #555; margin-bottom: 20px;}
        fieldset { border: 1px solid #ddd; padding: 15px; margin-bottom: 20px; border-radius: 6px; }
        legend { font-weight: bold; color: #007bff; padding: 0 10px; }
        label { display: block; margin-top: 10px; font-weight: bold; color: #333; }
        input[type="text"], input[type="number"], input[type="date"] { width: 95%; padding: 10px; margin-top: 5px; border: 1px solid #ccc; border-radius: 4px; }
        button { background-color: #28a745; color: white; padding: 12px 20px; border: none; border-radius: 4px; cursor: pointer; width: 100%; margin-top: 20px; font-size: 16px; transition: background-color 0.3s; }
        button:hover { background-color: #218838; }
        textarea { width: 95%; height: 150px; padding: 10px; margin-top: 15px; border: 1px solid #ccc; border-radius: 4px; background-color: #eee; }
        .error { color: red; margin-top: 10px; font-weight: bold; }
        .note { margin-top: 15px; padding: 10px; background-color: #e9f5ff; border: 1px solid #cce5ff; border-radius: 4px; font-size: 0.9em; }
    </style>
</head>
<body>
    <div class="container">
        <h2>🇮🇳 5-Minute GST E-Invoice Generator</h2>
        <p class="subtitle">Generate compliant JSON for a single-item B2B Service Invoice. Avoid common errors!</p>

        <form id="invoiceForm">
            <fieldset>
                <legend>Your (Supplier) Details</legend>
                <label for="supGstin">Your GSTIN:</label>
                <input type="text" id="supGstin" required maxlength="15" placeholder="e.g., 07AAAAA0000A1Z5 (Delhi)">
            </fieldset>

            <fieldset>
                <legend>Client (Recipient) Details</legend>
                <label for="recGstin">Client GSTIN:</label>
                <input type="text" id="recGstin" required maxlength="15" placeholder="e.g., 27BBBBB0000B1Z6 (Maharashtra)">
            </fieldset>

            <fieldset>
                <legend>Invoice & Service Details</legend>
                <label for="docNo">Invoice Number:</label>
                <input type="text" id="docNo" required maxlength="16" value="INV/FY25/001">

                <label for="docDate">Invoice Date (YYYY-MM-DD):</label>
                <input type="date" id="docDate" required value="2025-10-31">

                <label for="hsnCode">Service HSN Code:</label>
                <input type="text" id="hsnCode" required maxlength="8" value="998313" placeholder="e.g., 998313 (Graphic Design)">

                <label for="taxableValue">Taxable Value (₹):</label>
                <input type="number" id="taxableValue" required min="1" step="0.01" value="10000">

                <label for="gstRate">GST Rate (%):</label>
                <input type="number" id="gstRate" required min="0" max="28" step="0.01" value="18">
            </fieldset>
            
            <div class="note">
                **Testing Tip:** Use **07** for Delhi (Intra-state) and **27** for Maharashtra (Inter-state) as the first two digits of the GSTIN to test the CGST/SGST vs. IGST calculation logic.
            </div>

            <button type="button" onclick="generateJson()">Generate JSON File</button>
            <div id="errorMsg" class="error"></div>
        </form>

        <textarea id="jsonOutput" readonly placeholder="Your compliant JSON will appear here..."></textarea>
    </div>

    <script>
        // Mapping of State Codes (First two digits of GSTIN)
        const stateCodes = {
            "07": "DELHI", 
            "27": "MAHARASHTRA" 
            // Add more state codes if needed for production
        };

        function getGstinStateCode(gstin) {
            // Extracts the first two digits (State Code) from a GSTIN
            return gstin.substring(0, 2);
        }

        function generateJson() {
            document.getElementById('errorMsg').innerHTML = '';
            
            // 1. Fetch Inputs
            const supGstin = document.getElementById('supGstin').value.toUpperCase();
            const recGstin = document.getElementById('recGstin').value.toUpperCase();
            const docNo = document.getElementById('docNo').value;
            const docDateInput = document.getElementById('docDate').value;
            const hsnCode = document.getElementById('hsnCode').value;
            const taxableValue = parseFloat(document.getElementById('taxableValue').value);
            const gstRate = parseFloat(document.getElementById('gstRate').value);

            // Basic Validation
            if (supGstin.length !== 15 || recGstin.length !== 15) {
                document.getElementById('errorMsg').innerHTML = 'Error: GSTINs must be 15 characters long.';
                return;
            }
            if (supGstin === recGstin) {
                 document.getElementById('errorMsg').innerHTML = 'Error 2211: Supplier and Recipient GSTINs cannot be the same.';
                return;
            }
            if (isNaN(taxableValue) || isNaN(gstRate) || taxableValue <= 0) {
                document.getElementById('errorMsg').innerHTML = 'Error: Please enter valid Taxable Value and GST Rate.';
                return;
            }

            // 2. Calculate Taxes based on state code
            const supStateCode = getGstinStateCode(supGstin);
            const recStateCode = getGstinStateCode(recGstin);
            const isIntraState = supStateCode === recStateCode;
            
            const taxAmount = (taxableValue * gstRate) / 100;
            const totalInvoiceValue = taxableValue + taxAmount;

            let cgstAmt = 0;
            let sgstAmt = 0;
            let igstAmt = 0;

            if (isIntraState) {
                // Intra-state (CGST + SGST) - Avoids Error 2172
                cgstAmt = taxAmount / 2;
                sgstAmt = taxAmount / 2;
            } else {
                // Inter-state (IGST)
                igstAmt = taxAmount;
            }
            
            // Format Document Date to DD/MM/YYYY for IRP
            const docDateFormatted = docDateInput.split('-').reverse().join('/'); 

            // 3. Construct the Minimal Compliant JSON Structure (e-Invoice Schema)
            const jsonObject = {
                "TransDtls": {
                    "TaxSch": "GST",
                    "SupTyp": "B2B"
                },
                "DocumentDtls": {
                    "Typ": "INV",
                    "No": docNo,
                    "Date": docDateFormatted 
                },
                // Assuming basic legal and trade names for this MVP
                "SellerDtls": { 
                    "Gstin": supGstin,
                    "LglNm": "SUPPLIER NAME (Demo)", 
                    "Addr1": "Address Line 1",
                    "Loc": stateCodes[supStateCode] || "City",
                    "StCd": supStateCode,
                    "Pin": "110001" 
                },
                "BuyerDtls": {
                    "Gstin": recGstin,
                    "LglNm": "RECIPIENT NAME (Demo)",
                    "Addr1": "Client Address Line 1",
                    "Loc": stateCodes[recStateCode] || "Client City",
                    "StCd": recStateCode,
                    "Pin": "400001" 
                },
                "ItemList": [
                    {
                        "SlNo": "1",
                        "PrdDesc": "Professional Consulting Service Fee", 
                        "HsnCd": hsnCode,
                        "Qty": "1",
                        "Unit": "OTH", // Other
                        "UnitPrice": taxableValue.toFixed(2),
                        "TotAmt": taxableValue.toFixed(2),
                        "AssAmt": taxableValue.toFixed(2), 
                        "GstRt": gstRate.toFixed(2),
                        "IgstAmt": igstAmt.toFixed(2),
                        "CgstAmt": cgstAmt.toFixed(2),
                        "SgstAmt": sgstAmt.toFixed(2),
                        "TotItemVal": totalInvoiceValue.toFixed(2)
                    }
                ],
                "ValDtls": {
                    "AssValue": taxableValue.toFixed(2),
                    "TotInvVal": totalInvoiceValue.toFixed(2),
                    "CgstVal": cgstAmt.toFixed(2),
                    "SgstVal": sgstAmt.toFixed(2),
                    "IgstVal": igstAmt.toFixed(2)
                }
            };
            
            // 4. Output and Download
            const jsonString = JSON.stringify(jsonObject, null, 2);
            document.getElementById('jsonOutput').value = jsonString;

            // Optional: Auto-download the file
            const blob = new Blob([jsonString], { type: 'application/json' });
            const url = URL.createObjectURL(blob);
            const a = document.createElement('a');
            a.href = url;
            a.download = `${docNo.replace(/[^a-zA-Z0-9]/g, '_')}_eInvoice.json`;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        }
    </script>
</body>
</html>
