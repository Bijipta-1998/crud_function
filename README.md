# crud_function

<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet"/>
  <style>
    body {
      background-color: #f8f9fa;
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    }
    .form-box {
      background: #fff;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      padding: 24px;
      margin-bottom: 32px;
    }
    .record-table-wrapper {
      background: #fff;
      border-radius: 12px;
      box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
      padding: 24px;
      width: 100%;
      overflow-x: auto;
    }
    table {
      width: 100%;
      table-layout: auto;
      word-wrap: break-word;
      font-size: 0.9rem;
    }
    th, td {
      white-space: nowrap;
    }
    @media (max-width: 768px) {
      th, td {
        white-space: normal;
        font-size: 0.8rem;
      }
      .input-group {
        flex-direction: column;
      }
      .input-group .form-control {
        width: 100%;
        margin-bottom: 8px;
      }
      .input-group .btn {
        width: 100%;
      }
    }
  </style>
</head>
<body class="py-4">

  <div class="container">
    <div class="form-box">
      <h2 class="mb-4 text-center">Material Transfer Form</h2>
      <div class="row g-3 mb-3">
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">Client Name</label>
          <select id="client" class="form-select" onchange="updateLocationDropdown()">
            <option value="">-- Select Client --</option>
          </select>
        </div>
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">Location</label>
          <select id="location" class="form-select" onchange="updateFromDropdown()">
            <option value="">-- Select Location --</option>
          </select>
        </div>
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">From Location</label>
          <select id="from" class="form-select">
            <option value="">-- Select From Location --</option>
          </select>
        </div>
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">To Location</label>
          <select id="to" class="form-select">
            <option value="">-- Select To Location --</option>
          </select>
        </div>
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">Transferred Qty(In KG)</label>
          <input id="qty" type="number" class="form-control" placeholder="Enter quantity">
        </div>
        <div class="col-12 col-sm-6 col-md-4">
          <label class="form-label">Product</label>
          <select id="product" class="form-select">
            <option value="">-- Select Product --</option>
          </select>
        </div>
      </div>
      <div class="text-center mt-4">
        <button class="btn btn-primary me-2" onclick="submitData()">Submit</button>
        <button class="btn btn-secondary" onclick="resetForm()">Reset</button>
      </div>
    </div>
    <div class="form-box">
      <h4 class="mb-3 text-center">Search by ID</h4>
      <div class="input-group">
        <input id="searchId" type="text" class="form-control" placeholder="Enter ID">
        <button class="btn btn-outline-secondary" onclick="searchById()">Search</button>
      </div>
    </div>
    <div class="record-table-wrapper">
      <h4 class="mb-3 text-center">Transfer Records</h4>
      <div id="recordTable"></div>
    </div>
  </div>

  <script>

    let appData = {};
    let currentId = null;

    function initForm() {
      console.log("Initializing form...");
      google.script.run.withSuccessHandler(data => {
        console.log("Loaded data from Apps Script:", data);

        if (!data || typeof data !== 'object') {
          console.error("Data is null or not an object:", data);
          alert("Failed to load form data from server.");
          return;
        }

        appData = data;

        const clients = Object.keys(appData.data || {});
        const allStock = appData.allStockLocations || [];
        const records = appData.records || [];

        console.log("Clients available:", clients);
        console.log("All stock locations:", allStock);
        console.log("Records fetched:", records.length);

        populateDropdown("client", clients);
        populateDropdown("to", allStock);
        renderTable(records);

        // Populate the product dropdown independently
        populateProductDropdown();
      }).getSheetData();
    }

    function populateDropdown(id, options) {
      const dropdown = document.getElementById(id);
      dropdown.innerHTML = `<option value="">-- Select --</option>`;
      options.forEach(opt => {
        const o = document.createElement("option");
        o.value = opt;
        o.textContent = opt;
        dropdown.appendChild(o);
      });
      console.log(`Dropdown [${id}] populated with:`, options);
    }

    function populateProductDropdown() {
      const products = [];
      Object.keys(appData.data).forEach(client => {
        Object.keys(appData.data[client]).forEach(location => {
          Object.keys(appData.data[client][location]).forEach(stock => {
            appData.data[client][location][stock].forEach(product => {
              if (!products.includes(product)) {
                products.push(product);
              }
            });
          });
        });
      });

      populateDropdown("product", products);
    }

    function updateLocationDropdown() {
      const client = document.getElementById("client").value;
      console.log("Client selected:", client);
      const locations = client && appData.data[client] ? Object.keys(appData.data[client]) : [];
      populateDropdown("location", locations);
      document.getElementById("from").innerHTML = `<option value="">-- Select From Location --</option>`;
    }

    function updateFromDropdown() {
      const client = document.getElementById("client").value;
      const location = document.getElementById("location").value;
      const stocks = (appData.data[client] && appData.data[client][location]) || [];
      console.log(`From location for ${client} > ${location}:`, stocks);

      // Update 'from' dropdown based on selected client and location
      const fromLocations = Object.keys(stocks);
      populateDropdown("from", fromLocations);
    }

    function submitData() {
      const record = {
        id: currentId,
        client: document.getElementById("client").value,
        location: document.getElementById("location").value,
        from: document.getElementById("from").value,
        to: document.getElementById("to").value,
        qty: document.getElementById("qty").value,
        product: document.getElementById("product").value // Added product field
      };
      console.log("Submitting record:", record);
      google.script.run.withSuccessHandler(records => {
        console.log("Submit success. Updated records:", records);
        resetForm();
        renderTable(records);
      }).submitForm(record);
    }

    function renderTable(records) {
      const table = document.getElementById("recordTable");
      if (!records || !records.length) {
        table.innerHTML = "<p>No records available</p>";
        return;
      }

      const latestThree = records.slice(-3); // get last 3 records

      let html = `
        <table class="table table-bordered">
          <thead>
            <tr>
              <th>Unique ID</th>
              <th>Time Stamp</th>
              <th>Client</th>
              <th>Location</th>
              <th>From Location</th>
              <th>To Location</th>
              <th>Transferred Qty</th>
              <th>Product</th>
              <th>Action</th>
            </tr>
          </thead>
          <tbody>
      `;

      latestThree.forEach(r => {
        const time = new Date(r["Time Stime"]).toLocaleString(); // Format timestamp

        html += `
          <tr>
            <td>${r["Unique Id"] || ""}</td>
            <td>${time || ""}</td>
            <td>${r["Client"] || ""}</td>
            <td>${r["Location"] || ""}</td>
            <td>${r["From Location"] || ""}</td>
            <td>${r["To Location"] || ""}</td>
            <td>${r["Transfered QTY"] || ""}</td>
            <td>${r["Product"] || ""}</td> <!-- Display product -->
            <td>
              <button class="btn btn-sm btn-warning" onclick="editRecord('${r["Unique Id"]}')">Update</button>
              <button class="btn btn-sm btn-danger" onclick="deleteRecord('${r["Unique Id"]}')">Delete</button>
            </td>
          </tr>
        `;
      });

      html += "</tbody></table>";
      table.innerHTML = html;
    }

    function resetForm() {
      document.getElementById("client").value = "";
      document.getElementById("location").value = "";
      document.getElementById("from").value = "";
      document.getElementById("to").value = "";
      document.getElementById("qty").value = "";
      document.getElementById("product").value = ""; // Reset product dropdown
      currentId = null;
    }

    function searchById() {
      const id = document.getElementById("searchId").value;
      console.log("Searching for ID:", id);
      google.script.run.withSuccessHandler(records => {
        renderTable(records);
      }).searchById(id);
    }

    function editRecord(id) {
      console.log("Editing record with ID:", id);
      currentId = id;
      google.script.run.withSuccessHandler(record => {
        document.getElementById("client").value = record["Client"];
        document.getElementById("location").value = record["Location"];
        document.getElementById("from").value = record["From Location"];
        document.getElementById("to").value = record["To Location"];
        document.getElementById("qty").value = record["Transfered QTY"];
        document.getElementById("product").value = record["Product"];
      }).getRecordById(id);
    }

    function deleteRecord(id) {
      console.log("Deleting record with ID:", id);
      google.script.run.withSuccessHandler(records => {
        renderTable(records);
      }).deleteRecord(id);
    }

    window.onload = initForm;
  </script>
</body>
</html>
