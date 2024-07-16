<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <!-- <title>SQL to XLSX Converter</title> -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.17.0/xlsx.full.min.js"></script>
    <!-- <link rel="stylesheet" href="styles.css"> -->
    

</head>
<body>
    <h1 style="color: white;">SQL to XLSX Converter</h1>
    <input type="file" id="fileInput" style="color:aliceblue" />
    <button onclick="processFile()" style="padding: 10px 20px; background-color: #007bff; color: white; border: none; border-radius: 5px; cursor: pointer;">Convert to XLSX</button>

    <script>
        const processFile = () => {
            const fileInput = document.getElementById('fileInput');
            const file = fileInput.files[0];

            if (file) {
                const reader = new FileReader();
                reader.onload = (event) => {
                    const sqlContent = event.target.result;
                    processSQL(sqlContent);
                };
                reader.readAsText(file);
            } else {
                alert("Please select a file.");
            }
        };

        const processSQL = (sqlContent) => {
            const extractDataFromSQL = (content) => {
                const data = [];
                const lines = content.split('\n');
                let headers = null;
                let valuesString = "";

                lines.forEach(line => {
                    line = line.trim();
                    if (line.startsWith('INSERT INTO')) {
                        const headerMatch = line.match(/\(([^)]+)\)/);
                        if (headerMatch) {
                            headers = headerMatch[1].replace(/`/g, '').split(',').map(header => header.trim());
                            if (data.length === 0) {
                                data.push(headers); // Add headers only once
                            }
                        }

                        // Extract the values string after 'VALUES'
                        const valuesPart = line.substring(line.indexOf('VALUES') + 6).trim();
                        if (valuesString) {
                            valuesString += "," + valuesPart;
                        } else {
                            valuesString = valuesPart;
                        }
                    } else if (line.endsWith(");")) {
                        valuesString += line;
                    } else if (line.startsWith("(") || valuesString) {
                        // Continue appending multi-line values
                        valuesString += " " + line;
                    }
                });

                // Process the accumulated valuesString
                const valueSets = valuesString.split(/(?<=\)),\s*(?=\()/);

                valueSets.forEach(valueSet => {
                    const values = valueSet.replace(/^\(|\)$/g, '').split(/,\s*(?=(?:[^']*'[^']*')*[^']*$)/).map(value => value.trim().replace(/'/g, ''));
                    data.push(values);
                });

                console.log('Extracted data:', data); // Debug: log the extracted data

                return data;
            };

            const data = extractDataFromSQL(sqlContent);
            generateXLSX(data);
        };

        const generateXLSX = (data) => {
            const ws = XLSX.utils.aoa_to_sheet(data);
            const wb = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(wb, ws, "Sheet1");
            XLSX.writeFile(wb, 'converted.xlsx');
        };
    </script>
</body>
</html>
