<!DOCTYPE html>
<html lang="fa">
<head>
<meta charset="UTF-8">
<title>📡 High-Gain Antenna Pointing – Voyager</title>
<style>
html,body{margin:0;overflow:hidden;background:#000015;font-family:system-ui}
canvas{display:block}
.hud{
 position:absolute;left:16px;top:16px;
 padding:14px 18px;border-radius:14px;
 background:rgba(255,255,255,0.06);
 backdrop-filter:blur(10px);
 color:#d9f3ff;font-size:13px;min-width:430px;
}
.good{color:#9ff0ff}
.mid{color:#ffd29f}
.ad{color:#ff9f9f}
.warn{color:#ffb366}
</style>
</head>
<body>

<canvas id="c"></canvas>
<div class="hud" id="hud"></div>

<script>
const c=document.getElementById("c");
const ctx=c.getContext("2d");
let w,h;
function resize(){w=c.width=innerWidth;h=c.height=innerHeight}
resize();addEventListener("resize",resize);

const AU=65;
const sun={x:w/2,y:h/2};
const earth={a:AU*1,angle:0,r:6};
const heliopause=120;

const ship={
 distance:AU*1,
 angle:0.4,
 velocity:0.22,
 antennaAngle:0,
 beamWidth:6*Math.PI/180, // 6 degrees
 jitter:0,
 buffer:0
};

const txPower=220;

// Helpers
function pos(a,ang){
 return {x:sun.x+Math.cos(ang)*a,y:sun.y+Math.sin(ang)*a};
}

// Radio model with antenna gain
function link(distAU,pointError){
 let baseSignal=txPower/(distAU*distAU*160);

 // Directional gain model
 let gain=Math.max(0,1-(pointError/ship.beamWidth));
 if(pointError>ship.beamWidth) gain*=0.05;

 let signal=baseSignal*gain;
 let noise=0.12+Math.random()*0.05;
 if(distAU>heliopause) noise+=0.3;

 return {snr:signal/noise,latency:distAU*8.3,gain};
}

// Update
function update(){
 earth.angle+=0.0012;
 ship.distance+=ship.velocity;
 ship.angle+=0.002;

 // Compute pointing angle toward Earth
 const pe=pos(earth.a,earth.angle);
 const pship={
   x:sun.x+Math.cos(ship.angle)*ship.distance,
   y:sun.y+Math.sin(ship.angle)*ship.distance
 };

 let desired=Math.atan2(pe.y-pship.y,pe.x-pship.x);

 // Add small drift/jitter
 ship.jitter+=(Math.random()-0.5)*0.002;
 ship.antennaAngle+= (desired-ship.antennaAngle)*0.05 + ship.jitter;

 return {pe,pship,desired};
}

// Draw
function draw(){
 ctx.fillStyle="rgba(0,0,20,0.4)";
 ctx.fillRect(0,0,w,h);

 const {pe,pship,desired}=update();

 // Sun
 ctx.fillStyle="#ffcc66";
 ctx.beginPath();ctx.arc(sun.x,sun.y,12,0,Math.PI*2);ctx.fill();

 // Earth
 ctx.fillStyle="#2a7fff";
 ctx.beginPath();ctx.arc(pe.x,pe.y,6,0,Math.PI*2);ctx.fill();

 // Ship
 ctx.fillStyle="#fff";
 ctx.beginPath();ctx.arc(pship.x,pship.y,4,0,Math.PI*2);ctx.fill();

 // Draw antenna direction
 ctx.strokeStyle="cyan";
 ctx.beginPath();
 ctx.moveTo(pship.x,pship.y);
 ctx.lineTo(pship.x+Math.cos(ship.antennaAngle)*40,
            pship.y+Math.sin(ship.antennaAngle)*40);
 ctx.stroke();

 // True Earth direction
 ctx.strokeStyle="rgba(255,255,255,0.2)";
 ctx.beginPath();
 ctx.moveTo(pship.x,pship.y);
 ctx.lineTo(pship.x+Math.cos(desired)*40,
            pship.y+Math.sin(desired)*40);
 ctx.stroke();

 // Link calculation
 const distAU=Math.hypot(pship.x-pe.x,pship.y-pe.y)/AU;
 const pointError=Math.abs(desired-ship.antennaAngle);
 const q=link(distAU,pointError);

 let cls=q.snr>3?"good":q.snr>1?"mid":"bad";

 ctx.strokeStyle=cls==="good"?"rgba(120,220,255,0.5)":
                 cls==="mid" ?"rgba(255,200,120,0.5)":
                              "rgba(255,120,120,0.5)";
 ctx.beginPath();
 ctx.moveTo(pe.x,pe.y);
 ctx.lineTo(pship.x,pship.y);
 ctx.stroke();

 // HUD
 document.getElementById("hud").innerHTML=`
📡 High-Gain Antenna Control<br>
📏 Distance: ${distAU.toFixed(1)} AU<br>
🎯 Point Error: ${(pointError*180/Math.PI).toFixed(2)}°<br>
📊 Antenna Gain: ${q.gain.toFixed(2)}<br>
📶 SNR: <span class="${cls}">${q.snr.toFixed(2)}</span><br>
🕒 Latency: ${(q.latency/60).toFixed(2)} h<br>
📡 Link: <span class="${cls}">
${q.snr>0.8?'LOCKED':'MISALIGNED'}
</span>
`;

 requestAnimationFrame(draw);
}

draw();
</script>
</body>
</html>
