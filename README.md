<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
<title>8-Ball Player Stats Web App</title>
<!-- Load external libraries: Chart.js for graphs, PapaParse for CSV parsing, XLSX for Excel export -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdn.jsdelivr.net/npm/papaparse@5.4.1/papaparse.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js"></script>
<style>
  body { font-family: Arial, sans-serif; margin: 20px; }
  table { width:100%; border-collapse:collapse; margin-top:10px; }
  th,td { border:1px solid #ccc; padding:4px; text-align:center; }
  button { margin-top: 5px; }
</style>
</head>
<body>
<h1>🎱 8-Ball Player Stats</h1>
<!-- File upload -->
<input type="file" id="fileInput" accept=".csv, application/vnd.openxmlformats-officedocument.spreadsheetml.sheet, application/vnd.ms-excel" />
<button onclick="uploadFile()">Upload File</button>

<!-- Manual form to add player -->
<h3>Add Player</h3>
<input placeholder="Name" id="name" />
<select id="comp" onchange="toggleDivision()">
  <option value="Winter">Winter</option>
  <option value="Summer">Summer</option>
</select>
<select id="division">
  <option value="A">Division A</option>
  <option value="B">Division B</option>
</select>
<input type="number" placeholder="Total Frames" id="matches" />
<input type="number" placeholder="Frames Won" id="framesWon" />
<input type="number" placeholder="Frames Lost" id="framesLost" />
<input type="number" placeholder="Master Break" id="masterBreak" />
<input type="number" placeholder="Master Shot" id="masterShot" />
<input type="number" placeholder="Master Clearance" id="masterClearance" />
<button onclick="addPlayer()">Add</button>

<!-- Display tables and charts for each comp/division -->
<h3>Winter Comp - Division A</h3>
<table id="tableWinterA"></table>
<canvas id="chartWinterA" width="400" height="150"></canvas>
<button onclick="downloadXLSX('Winter','A')">Download XLSX</button>

<h3>Winter Comp - Division B</h3>
<table id="tableWinterB"></table>
<canvas id="chartWinterB" width="400" height="150"></canvas>
<button onclick="downloadXLSX('Winter','B')">Download XLSX</button>

<h3>Summer Comp</h3>
<table id="tableSummer"></table>
<canvas id="chartSummer" width="400" height="150"></canvas>
<button onclick="downloadXLSX('Summer','')">Download XLSX</button>

<script>
// Array to hold all player data
let players=[];
// Chart instances
let chartWinterA, chartWinterB, chartSummer;

// Show/hide division dropdown based on selected comp
function toggleDivision(){
  const comp=document.getElementById('comp').value;
  document.getElementById('division').style.display=comp==='Winter'?'inline-block':'none';
}
toggleDivision();

// Handle file upload (CSV or Excel)
function uploadFile(){
  const file=document.getElementById('fileInput').files[0];
  if(!file)return;
  if(file.name.endsWith('.csv')){
    // Parse CSV file using PapaParse
    Papa.parse(file,{header:true,skipEmptyLines:true,complete:res=>processData(res.data)});
  }else{
    // Parse Excel file using XLSX
    const reader=new FileReader();
    reader.onload=e=>{
      const wb=XLSX.read(new Uint8Array(e.target.result),{type:'array'});
      processData(XLSX.utils.sheet_to_json(wb.Sheets[wb.SheetNames[0]]));
    };
    reader.readAsArrayBuffer(file);
  }
}

// Process parsed data, calculate average percentage, and store
function processData(data){
  data.forEach(p=>{
    p.Matches=+p.Matches||0;
    p['Frames Won']=+p['Frames Won']||0;
    p['Frames Lost']=+p['Frames Lost']||0;
    p['Master Break']=+p['Master Break']||0;
    p['Master Shot']=+p['Master Shot']||0;
    p['Master Clearance']=+p['Master Clearance']||0;
    p.Average=p.Matches?((p['Frames Won']/p.Matches)*100).toFixed(2):'0'; // Average as percentage
    players.push(p);
  });
  renderAll();
}

// Add player manually from form
function addPlayer(){
  const name=document.getElementById('name').value;
  const comp=document.getElementById('comp').value;
  const division=comp==='Winter'?document.getElementById('division').value:'';
  const m=+document.getElementById('matches').value||0;
  const fw=+document.getElementById('framesWon').value||0;
  const fl=+document.getElementById('framesLost').value||0;
  const mb=+document.getElementById('masterBreak').value||0;
  const ms=+document.getElementById('masterShot').value||0;
  const mc=+document.getElementById('masterClearance').value||0;
  const avg=m?((fw/m)*100).toFixed(2):'0'; // Calculate average as percentage
  players.push({Name:name,Comp:comp,Division:division,Matches:m,'Frames Won':fw,'Frames Lost':fl,'Master Break':mb,'Master Shot':ms,'Master Clearance':mc,Average:avg});
  renderAll();
}

// Render all tables and charts
function renderAll(){
  renderTableChart('Winter','A');
  renderTableChart('Winter','B');
  renderTableChart('Summer','');
}

// Render table and chart for specific comp/division
function renderTableChart(comp,division){
  let filtered=players.filter(p=>p.Comp==comp&&(comp!=='Winter'||p.Division==division));
  filtered.sort((a,b)=>b.Average-a.Average); // Sort descending by average

  // Build HTML table
  let html='<tr><th>Name</th><th>Total Frames</th><th>Frames Won</th><th>Frames Lost</th><th>Average (%)</th><th>Master Break</th><th>Master Shot</th><th>Master Clearance</th></tr>';
  filtered.forEach(p=>{
    html+=`<tr><td>${p.Name}</td><td>${p.Matches}</td><td>${p['Frames Won']}</td><td>${p['Frames Lost']}</td><td>${p.Average}</td><td>${p['Master Break']}</td><td>${p['Master Shot']}</td><td>${p['Master Clearance']}</td></tr>`;
  });

  // Insert table into DOM
  let tid=comp=='Summer'?'tableSummer':(division=='A'?'tableWinterA':'tableWinterB');
  document.getElementById(tid).innerHTML=html;

  // Prepare chart context and destroy old chart if needed
  let ctxid=comp=='Summer'?'chartSummer':(division=='A'?'chartWinterA':'chartWinterB');
  const ctx=document.getElementById(ctxid).getContext('2d');
  if((comp=='Winter'&&division=='A'&&chartWinterA))chartWinterA.destroy();
  if((comp=='Winter'&&division=='B'&&chartWinterB))chartWinterB.destroy();
  if(comp=='Summer'&&chartSummer)chartSummer.destroy();

  // Create new bar chart if data exists
  if(filtered.length){
    const newChart=new Chart(ctx,{type:'bar',data:{labels:filtered.map(p=>p.Name),datasets:[{label:'Average (%)',data:filtered.map(p=>+p.Average)}]},options:{scales:{y:{beginAtZero:true}}}});
    if(comp=='Winter'&&division=='A')chartWinterA=newChart;
    if(comp=='Winter'&&division=='B')chartWinterB=newChart;
    if(comp=='Summer')chartSummer=newChart;
  }
}

// Download filtered data as XLSX file
function downloadXLSX(comp,division){
  let filtered=players.filter(p=>p.Comp==comp&&(comp!=='Winter'||p.Division==division));
  const ws=XLSX.utils.json_to_sheet(filtered);
  const wb=XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb,ws,'Stats');
  XLSX.writeFile(wb,`${comp}_${division||'All'}.xlsx`);
}
</script>
</body>
</html>
