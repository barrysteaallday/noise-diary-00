<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Noise Diary PWA</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>
body{
    margin:0;
    padding:20px;
    background:#F5EFEC;
    font-family:"Georgia","Times New Roman",serif;
    color:#333;
}

.container{
    max-width:900px;
    margin:auto;
}

h1{
    text-align:center;
    font-weight:900;
    font-size:34px;
    letter-spacing:1px;
}

/* FORM */
.form-box{
    background:#EDE8E5;
    padding:20px;
    border-radius:12px;
    border:1px solid #ddd;
}

label{
    display:block;
    margin-top:10px;
    font-weight:700;
}

input,select,textarea{
    width:100%;
    padding:10px;
    margin-top:5px;
    border:1px solid #ccc;
    font-family:inherit;
}

/* BUTTON */
button{
    width:100%;
    margin-top:12px;
    padding:12px;
    background:#777;
    color:#F5EFEC;
    border:none;
    font-weight:700;
    cursor:pointer;
}

/* ENTRY BOX */
.entry{
    background:#F5EFEC;
    margin-top:10px;
    padding:12px;
    border-radius:8px;
    border:1px solid #ddd;
    font-weight:400;
    font-size:15px;
    line-height:1.5;
}

/* SUBTLE EDIT/DELETE BUTTONS */
.actions{
    margin-top:8px;
    opacity:0.15;
    font-size:11px;
    display:flex;
    gap:6px;
}

.actions button{
    width:auto;
    padding:2px 6px;
    font-size:10px;
    background:transparent;
    color:#666;
    border:1px solid #ccc;
    border-radius:4px;
    cursor:pointer;
    transition:0.2s;
}

.actions button:hover{
    opacity:1;
    background:#EDE8E5;
}

/* GRAPH + BOXES */
canvas,#summaryBox,#letterBox{
    margin-top:15px;
    background:#EDE8E5;
    border:1px solid #ddd;
    padding:12px;
    border-radius:8px;
    white-space:pre-line;
}

/* SMALL LABEL */
.small{
    font-size:12px;
    opacity:0.7;
}
</style>
</head>

<body>

<div class="container">

<h1>Noise Diary</h1>

<div class="form-box">
<label>Date</label>
<input type="date" id="date">
<label>Time (24-hour)</label>
<input type="time" id="time" step="60">
<label>Duration</label>
<select id="duration">
<option value="">Select</option>
<option>1 min</option>
<option>2 min</option>
<option>3 min</option>
<option>4 min</option>
<option>5 min</option>
<option>10 min</option>
<option>15 min</option>
<option>20 min</option>
<option>30 min</option>
</select>
<label>Notes</label>
<textarea id="notes" placeholder="Describe disturbance, effect on sleep, etc."></textarea>
<button onclick="addEntry()">Add Entry</button>
</div>

<h2>Entries</h2>
<div id="log"></div>

<h2>Analysis (Line Graph)</h2>
<canvas id="graph"></canvas>

<h2>Summary</h2>
<div id="summaryBox"></div>

<button onclick="generateLetter()">Generate Council Letter</button>
<div id="letterBox"></div>

</div>

<script>
let entries = JSON.parse(localStorage.getItem("noiseDiary") || "[]");
let chart;

/* AUTO DATE + TIME */
function setAutoDateTime(){
    const now=new Date();
    const yyyy=now.getFullYear();
    const mm=String(now.getMonth()+1).padStart(2,'0');
    const dd=String(now.getDate()).padStart(2,'0');
    const hh=String(now.getHours()).padStart(2,'0');
    const min=String(now.getMinutes()).padStart(2,'0');
    document.getElementById("date").value=`${yyyy}-${mm}-${dd}`;
    document.getElementById("time").value=`${hh}:${min}`;
}

/* SAVE */
function save(){
    localStorage.setItem("noiseDiary",JSON.stringify(entries));
}

/* SORT */
function getDT(e){
    return new Date(e.date+"T"+e.time);
}

/* SCORE */
function getScore(e){
    let score=1;
    if(["2","3","4","5"].some(x=>e.duration.startsWith(x))) score=2;
    if(["6","7","8","9","10"].some(x=>e.duration.startsWith(x))) score=3;
    if(["15","20","30"].some(x=>e.duration.startsWith(x))) score=4;
    if(e.time>="00:00"&&e.time<="03:00") score+=2;
    return score;
}

/* RENDER */
function render(){
    const log=document.getElementById("log");
    log.innerHTML="";
    const sorted=[...entries].sort((a,b)=>getDT(b)-getDT(a));
    sorted.forEach((e,i)=>{
        const div=document.createElement("div");
        div.className="entry";
        const sleep=(e.time>="00:00"&&e.time<="03:00")?" night disturbance":"";
        div.innerHTML=`
        <div>
            <b>Date:</b> ${e.date} |
            <b>Time:</b> ${e.time} |
            <b>Duration:</b> ${e.duration}${sleep}<br>
            <b>Notes:</b> ${e.notes}
        </div>
        <div class="actions">
            <button onclick="editEntry(${i})">✎</button>
            <button onclick="deleteEntry(${i})">✕</button>
        </div>
        `;
        log.appendChild(div);
    });
    drawGraph(sorted);
    updateSummary(sorted);
}

/* ADD */
function addEntry(){
    const date=document.getElementById("date").value;
    const time=document.getElementById("time").value;
    const duration=document.getElementById("duration").value;
    const notes=document.getElementById("notes").value.trim();
    if(!date||!time||!duration||!notes){
        alert("Complete all fields");
        return;
    }
    entries.push({date,time,duration,notes});
    save();
    render();
    setAutoDateTime();
    document.getElementById("notes").value="";
    document.getElementById("duration").value="";
}

/* ENTER KEY SUBMIT */
document.getElementById("notes").addEventListener("keydown",function(e){
    if(e.key==="Enter"){
        e.preventDefault();
        addEntry();
    }
});

/* DELETE */
function deleteEntry(i){
    entries.splice(i,1);
    save();
    render();
}

/* EDIT */
function editEntry(i){
    const e=entries[i];
    const d=prompt("Date",e.date);
    const t=prompt("Time",e.time);
    const du=prompt("Duration",e.duration);
    const n=prompt("Notes",e.notes);
    if(d&&t&&du&&n){
        entries[i]={date:d,time:t,duration:du,notes:n};
        save();
        render();
    }
}

/* GRAPH */
function drawGraph(data){
    const ctx=document.getElementById("graph").getContext("2d");
    const counts={};
    data.forEach(e=>{counts[e.date]=(counts[e.date]||0)+1;});
    const labels=Object.keys(counts);
    const values=Object.values(counts);
    if(chart) chart.destroy();
    chart=new Chart(ctx,{
        type:"line",
        data:{
            labels,
            datasets:[{
                label:"Noise Events",
                data:values,
                borderColor:"#666",
                backgroundColor:"rgba(102,102,102,0.1)",
                fill:true,
                tension:0.3
            }]
        },
        options:{scales:{y:{beginAtZero:true}}}
    });
}

/* SUMMARY */
function updateSummary(data){
    let total=0;
    let night=0;
    data.forEach(e=>{total+=getScore(e);if(e.time>="00:00"&&e.time<="03:00") night++;});
    let level="low";
    if(total>10) level="moderate";
    if(total>20) level="high";
    document.getElementById("summaryBox").innerText=
`Total entries: ${data.length}
Night-time events: ${night}
Impact score: ${total}
Severity level: ${level}`;
}

/* LETTER */
function generateLetter(){
    if(entries.length===0){alert("No entries");return;}
    const sorted=[...entries].sort((a,b)=>getDT(a)-getDT(b));
    const dates=sorted.map(e=>`${e.date} at ${e.time}`).join(", ");
    const letter=`Formal Complaint – Ongoing Dog Noise Nuisance

I wish to lodge a formal complaint regarding persistent and unreasonable dog noise from my neighbour. Based on recorded events (${dates}), the disturbances regularly occurred during the night, affecting our sleep and work safety.

Attached is my noise diary as evidence.

I respectfully request immediate action to prevent further disturbance.`;
    document.getElementById("letterBox").innerText=letter;
}

/* INIT */
setAutoDateTime();
render();
</script>

</body>
</html>
