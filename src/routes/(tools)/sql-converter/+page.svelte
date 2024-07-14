<script>
	import { onMount } from 'svelte';
	import initSqlJs from 'sql.js';
	import * as XLSX from 'xlsx';
  
	let db;
	let error = '';
	let successMessage = '';
	let isFileSelected = false;
	let fileContent = '';
  
	onMount(async () => {
	  try {
		const SQL = await initSqlJs({
		  locateFile: file => `https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.5.0/sql-wasm.wasm`
		});
		db = new SQL.Database();
	  } catch (err) {
		error = `Error initializing SQL.js: ${err.message}`;
		console.error(error);
	  }
	});
  
	async function handleFileUpload(event) {
	  error = '';
	  successMessage = '';
	  const file = event.target.files[0];
	  if (!file) {
		error = 'No file selected';
		isFileSelected = false;
		return;
	  }
	  isFileSelected = true;
  
	  // Read the file content
	  const reader = new FileReader();
	  reader.onload = (e) => {
		fileContent = e.target.result;
	  };
	  reader.onerror = (e) => {
		error = `File reading error: ${e.target.error.message}`;
		console.error(error);
		isFileSelected = false;
	  };
	  reader.readAsText(file);
	}
  
	function preprocessSQL(sql) {
	  // Remove AUTO_INCREMENT and other MySQL-specific syntax
	  return sql
		.replace(/AUTO_INCREMENT/gi, '')
		.replace(/`/g, '') // Remove backticks
		.replace(/UNSIGNED/gi, ''); // Remove UNSIGNED
	}
  
	async function exportToXlsx() {
	  if (!isFileSelected) {
		error = 'No file selected';
		return;
	  }
	  error = '';
	  successMessage = '';
  
	  try {
		let sql = preprocessSQL(fileContent);
		db.run(sql);
  
		const tables = db.exec("SELECT name FROM sqlite_master WHERE type='table';");
		if (tables.length === 0) {
		  throw new Error('No tables found in the SQL file.');
		}
  
		const workbook = XLSX.utils.book_new();
		for (const table of tables[0].values) {
		  const tableName = table[0];
		  const data = db.exec(`SELECT * FROM ${tableName}`);
		  if (data.length === 0) continue;
		  const rows = data[0].values;
		  const headers = data[0].columns;
		  const worksheet = XLSX.utils.aoa_to_sheet([headers, ...rows]);
		  XLSX.utils.book_append_sheet(workbook, worksheet, tableName);
		}
  
		XLSX.writeFile(workbook, 'output.xlsx');
		successMessage = 'XLSX file generated successfully!';
	  } catch (err) {
		error = `Error: ${err.message}`;
		console.error(err);
	  }
	}
  </script>
  
  <style>
	.container {
	  text-align: center;
	  margin-top: 50px;
	}
	.button {
	  background-color: #007bff;
	  color: white;
	  padding: 10px 20px;
	  border: none;
	  border-radius: 5px;
	  cursor: pointer;
	}
	.button:hover {
	  background-color: #0056b3;
	}
	.message {
	  margin-top: 20px;
	  padding: 10px;
	  border-radius: 4px;
	  display: inline-block;
	}
	.error {
	  background-color: #f8d7da;
	  color: #721c24;
	}
	.success {
	  background-color: #d4edda;
	  color: #155724;
	}
  </style>
  
  <div class="container">
	<h1>SQL to XLSX Converter</h1>
	<input type="file" accept=".sql" on:change={handleFileUpload} />
	<br /><br />
	<button class="button" on:click={exportToXlsx}>Export to XLSX</button>
	<br /><br />
	{#if error}
	  <div class="message error">{error}</div>
	{/if}
	{#if successMessage}
	  <div class="message success">{successMessage}</div>
	{/if}
  </div>
