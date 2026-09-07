<!DOCTYPE html>
<html lang="id">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no">
<title>ESCAPE FROM SMA BAHAGIA</title>

<style>
*{
    box-sizing:border-box;
    user-select:none;
    -webkit-user-select:none;
    touch-action:none;
}

html,body{
    margin:0;
    width:100%;
    height:100%;
    overflow:hidden;
    background:#18202b;
    font-family:Arial,sans-serif;
}

canvas{
    display:block;
    width:100vw;
    height:100vh;
    background:#d9c79d;
}

#hud{
    position:fixed;
    top:10px;
    left:10px;
    right:10px;
    display:flex;
    justify-content:space-between;
    z-index:5;
    pointer-events:none;
}

.hudBox{
    background:rgba(20,25,32,.88);
    color:white;
    padding:9px 13px;
    border-radius:12px;
    font-weight:bold;
}

#message{
    position:fixed;
    top:62px;
    left:50%;
    transform:translateX(-50%);
    background:rgba(20,25,32,.9);
    color:white;
    padding:9px 15px;
    border-radius:12px;
    z-index:6;
    text-align:center;
    max-width:90%;
    font-size:14px;
}

#controls{
    position:fixed;
    bottom:18px;
    left:18px;
    right:18px;
    display:flex;
    justify-content:space-between;
    align-items:end;
    z-index:8;
}

.dpad{
    display:grid;
    grid-template-columns:55px 55px 55px;
    grid-template-rows:55px 55px 55px;
    gap:5px;
}

.btn{
    border:0;
    border-radius:14px;
    background:rgba(20,25,32,.8);
    color:white;
    font-size:22px;
    font-weight:bold;
    box-shadow:0 4px 8px rgba(0,0,0,.25);
}

.btn:active{
    transform:scale(.93);
}

.up{grid-column:2}
.left{grid-column:1;grid-row:2}
.down{grid-column:2;grid-row:3}
.right{grid-column:3;grid-row:2}

.run{
    width:78px;
    height:78px;
    border-radius:50%;
    font-size:16px;
    background:#b52c2c;
}

.overlay{
    position:fixed;
    inset:0;
    display:flex;
    justify-content:center;
    align-items:center;
    z-index:20;
    background:rgba(5,10,15,.78);
}

.card{
    width:min(90%,430px);
    background:#f4ead0;
    color:#20252c;
    padding:28px;
    border-radius:22px;
    text-align:center;
    box-shadow:0 15px 40px rgba(0,0,0,.4);
}

.card h1{
    margin-top:0;
    font-size:28px;
}

.card button{
    border:0;
    background:#31549b;
    color:white;
    padding:14px 25px;
    border-radius:12px;
    font-size:17px;
    font-weight:bold;
}

.hidden{
    display:none;
}
</style>
</head>

<body>

<canvas id="game"></canvas>

<div id="hud">
    <div class="hudBox">❤️ <span id="lives">3</span></div>
    <div class="hudBox">⭐ <span id="score">0</span></div>
    <div class="hudBox">🔑 <span id="keyText">BELUM</span></div>
</div>

<div id="message">Cari kunci lalu kabur lewat gerbang!</div>

<div id="controls">
    <div class="dpad">
        <button class="btn up" data-key="up">▲</button>
        <button class="btn left" data-key="left">◀</button>
        <button class="btn down" data-key="down">▼</button>
        <button class="btn right" data-key="right">▶</button>
    </div>

    <button class="btn run" data-key="run">RUN</button>
</div>

<div id="menu" class="overlay">
    <div class="card">
        <h1>🏫 ESCAPE FROM SMA BAHAGIA</h1>
        <p>
            Ambil 🔑 kunci, hindari guru dan satpam,
            lalu kabur lewat 🚪 gerbang sekolah!
        </p>
        <p>
            Gunakan tombol arah untuk bergerak.
            Tahan <b>RUN</b> untuk berlari.
        </p>
        <button onclick="startGame()">MULAI</button>
    </div>
</div>

<div id="gameover" class="overlay hidden">
    <div class="card">
        <h1>😭 KETAHUAN!</h1>
        <p>Guru atau satpam berhasil menangkapmu.</p>
        <button onclick="restartGame()">COBA LAGI</button>
    </div>
</div>

<div id="win" class="overlay hidden">
    <div class="card">
        <h1>🏆 BERHASIL KABUR!</h1>
        <p>Kamu berhasil keluar dari SMA BAHAGIA!</p>
        <p>⭐ Skor: <span id="finalScore">0</span></p>
        <button onclick="restartGame()">MAIN LAGI</button>
    </div>
</div>

<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");

let W=innerWidth;
let H=innerHeight;

function resize(){
    W=innerWidth;
    H=innerHeight;
    canvas.width=W;
    canvas.height=H;
}

addEventListener("resize",resize);
resize();

const WORLD_W=3200;
const WORLD_H=1800;

let cameraX=0;
let cameraY=0;

let gameRunning=false;
let gameWon=false;

let lives=3;
let score=0;
let damageCooldown=0;
let messageTimer=0;

const keys={
    up:false,
    down:false,
    left:false,
    right:false,
    run:false
};

const player={
    x:230,
    y:850,
    w:42,
    h:58,
    speed:3.8
};

const playerImg=new Image();
playerImg.src="assets/player.png";


// =========================
// WALLS
// =========================

const walls=[

    {x:0,y:0,w:3200,h:50},
    {x:0,y:1750,w:3200,h:50},
    {x:0,y:0,w:50,h:1800},
    {x:3150,y:0,w:50,h:1800},

    // KELAS X-A
    {x:120,y:200,w:650,h:35},
    {x:120,y:600,w:650,h:35},
    {x:120,y:200,w:35,h:435},
    {x:735,y:200,w:35,h:435},

    // KELAS X-B
    {x:850,y:200,w:600,h:35},
    {x:850,y:600,w:600,h:35},
    {x:850,y:200,w:35,h:435},
    {x:1415,y:200,w:35,h:435},

    // KELAS X-C
    {x:1530,y:200,w:650,h:35},
    {x:1530,y:600,w:650,h:35},
    {x:1530,y:200,w:35,h:435},
    {x:2145,y:200,w:35,h:435},

    // LAB
    {x:2250,y:200,w:650,h:35},
    {x:2250,y:600,w:650,h:35},
    {x:2250,y:200,w:35,h:435},
    {x:2865,y:200,w:35,h:435},

    // KELAS XI
    {x:120,y:1050,w:700,h:35},
    {x:120,y:1450,w:700,h:35},
    {x:120,y:1050,w:35,h:435},
    {x:785,y:1050,w:35,h:435},

    // KELAS XII
    {x:900,y:1050,w:700,h:35},
    {x:900,y:1450,w:700,h:35},
    {x:900,y:1050,w:35,h:435},
    {x:1565,y:1050,w:35,h:435},

    // TOILET
    {x:1670,y:1050,w:400,h:35},
    {x:1670,y:1450,w:400,h:35},
    {x:1670,y:1050,w:35,h:435},
    {x:2035,y:1050,w:35,h:435},

    // TANGGA
    {x:2140,y:1050,w:500,h:35},
    {x:2140,y:1450,w:500,h:35},
    {x:2140,y:1050,w:35,h:435},
    {x:2605,y:1050,w:35,h:435},

    // PEMBATAS KORIDOR
    {x:800,y:700,w:40,h:300},
    {x:1580,y:700,w:40,h:300},
    {x:2050,y:700,w:40,h:300},
    {x:2650,y:700,w:40,h:300}
];


// =========================
// FURNITURE
// =========================

const furniture=[

    // KELAS X-A
    {type:"desk",x:180,y:280},
    {type:"desk",x:280,y:280},
    {type:"desk",x:380,y:280},
    {type:"desk",x:480,y:280},
    {type:"desk",x:180,y:400},
    {type:"desk",x:280,y:400},
    {type:"desk",x:380,y:400},
    {type:"desk",x:480,y:400},
    {type:"teacherDesk",x:590,y:270},
    {type:"board",x:250,y:250},

    // X-B
    {type:"desk",x:900,y:280},
    {type:"desk",x:1000,y:280},
    {type:"desk",x:1100,y:280},
    {type:"desk",x:1200,y:280},
    {type:"desk",x:900,y:400},
    {type:"desk",x:1000,y:400},
    {type:"desk",x:1100,y:400},
    {type:"desk",x:1200,y:400},
    {type:"teacherDesk",x:1310,y:270},
    {type:"board",x:970,y:250},

    // X-C
    {type:"desk",x:1580,y:280},
    {type:"desk",x:1680,y:280},
    {type:"desk",x:1780,y:280},
    {type:"desk",x:1880,y:280},
    {type:"desk",x:1580,y:400},
    {type:"desk",x:1680,y:400},
    {type:"desk",x:1780,y:400},
    {type:"desk",x:1880,y:400},
    {type:"teacherDesk",x:2010,y:270},
    {type:"board",x:1650,y:250},

    // LAB
    {type:"labTable",x:2300,y:280},
    {type:"labTable",x:2500,y:280},
    {type:"labTable",x:2700,y:280},
    {type:"cabinet",x:2320,y:500},
    {type:"cabinet",x:2440,y:500},
    {type:"cabinet",x:2560,y:500},

    // BAWAH
    {type:"desk",x:190,y:1120},
    {type:"desk",x:290,y:1120},
    {type:"desk",x:390,y:1120},
    {type:"desk",x:490,y:1120},
    {type:"bench",x:180,y:1320},
    {type:"bench",x:330,y:1320},
    {type:"bench",x:480,y:1320},

    {type:"desk",x:970,y:1120},
    {type:"desk",x:1070,y:1120},
    {type:"desk",x:1170,y:1120},
    {type:"desk",x:1270,y:1120},
    {type:"bench",x:970,y:1320},
    {type:"bench",x:1120,y:1320},
    {type:"bench",x:1270,y:1320},

    {type:"cabinet",x:1740,y:1130},
    {type:"cabinet",x:1850,y:1130},

    {type:"bench",x:2200,y:1140},
    {type:"bench",x:2200,y:1250},
    {type:"bench",x:2200,y:1360},

    // KORIDOR
    {type:"plant",x:650,y:800},
    {type:"plant",x:1450,y:800},
    {type:"plant",x:2050,y:800},
    {type:"plant",x:2850,y:800},

    {type:"trash",x:700,y:900},
    {type:"trash",x:1500,y:900},
    {type:"trash",x:2100,y:900},
    {type:"trash",x:2800,y:900},

    {type:"bench",x:900,y:800},
    {type:"bench",x:1100,y:800},
    {type:"bench",x:1700,y:800},
    {type:"bench",x:2300,y:800}
];

function furnitureCollision(){

    return furniture.map(f=>{

        if(f.type==="desk")
            return {x:f.x,y:f.y,w:65,h:65};

        if(f.type==="teacherDesk")
            return {x:f.x,y:f.y,w:80,h:65};

        if(f.type==="labTable")
            return {x:f.x,y:f.y,w:120,h:75};

        if(f.type==="cabinet")
            return {x:f.x,y:f.y,w:50,h:70};

        if(f.type==="bench")
            return {x:f.x,y:f.y,w:110,h:40};

        return null;

    }).filter(Boolean);
}


// =========================
// GURU
// =========================

const enemies=[

    {
        x:500,y:760,w:42,h:55,
        vx:1.2,minX:300,maxX:700,
        state:"patrol",
        lastSeenX:0,lastSeenY:0,
        searchTimer:0,angle:0
    },

    {
        x:1200,y:760,w:42,h:55,
        vx:1.35,minX:900,maxX:1500,
        state:"patrol",
        lastSeenX:0,lastSeenY:0,
        searchTimer:0,angle:0
    },

    {
        x:1900,y:760,w:42,h:55,
        vx:1.3,minX:1650,maxX:2050,
        state:"patrol",
        lastSeenX:0,lastSeenY:0,
        searchTimer:0,angle:0
    }
];


// =========================
// SATPAM
// =========================

const guard={
    x:1510,
    y:1580,
    w:48,
    h:58,
    state:"patrol",
    vx:1.1,
    minX:1320,
    maxX:1700,
    lastSeenX:0,
    lastSeenY:0,
    searchTimer:0,
    angle:Math.PI
};


// =========================
// KUNCI RANDOM
// =========================

const key={
    x:2700,
    y:800,
    collected:false
};

const keySpawns=[

    {x:700,y:750},
    {x:1100,y:750},
    {x:1500,y:750},
    {x:2100,y:850},
    {x:2450,y:850},
    {x:2800,y:1200},
    {x:1300,y:1500},
    {x:2050,y:1500}
];

function randomizeKey(){

    const spot=
        keySpawns[
            Math.floor(
                Math.random()*keySpawns.length
            )
        ];

    key.x=spot.x;
    key.y=spot.y;
    key.collected=false;
}


// =========================
// GERBANG
// =========================

const gate={
    x:2700,
    y:1580,
    w:400,
    h:170,
    open:false
};

const exitZone={
    x:2820,
    y:1630,
    w:170,
    h:110
};


// =========================
// KOIN
// =========================

const coins=[

    {x:700,y:750,taken:false},
    {x:1550,y:750,taken:false},
    {x:2100,y:750,taken:false},
    {x:2450,y:850,taken:false},
    {x:2800,y:1200,taken:false},
    {x:1300,y:1500,taken:false},
    {x:2050,y:1500,taken:false}
];


// =========================
// UTILITAS
// =========================

function showMessage(text,time=120){

    document.getElementById("message")
        .textContent=text;

    messageTimer=time;
}

function rectsOverlap(a,b){

    return(
        a.x<b.x+b.w &&
        a.x+a.w>b.x &&
        a.y<b.y+b.h &&
        a.y+a.h>b.y
    );
}

function blocked(rect){

    for(const w of walls){

        if(rectsOverlap(rect,w))
            return true;

    }

    for(const f of furnitureCollision()){

        if(rectsOverlap(rect,f))
            return true;

    }

    if(!gate.open){

        const gateWall={
            x:gate.x,
            y:gate.y,
            w:gate.w,
            h:45
        };

        if(rectsOverlap(rect,gateWall))
            return true;
    }

    return false;
}


// =========================
// PLAYER
// =========================

function movePlayer(dx,dy){

    const nx={
        x:player.x+dx,
        y:player.y,
        w:player.w,
        h:player.h
    };

    if(!blocked(nx))
        player.x+=dx;

    const ny={
        x:player.x,
        y:player.y+dy,
        w:player.w,
        h:player.h
    };

    if(!blocked(ny))
        player.y+=dy;
}


// =========================
// LINE OF SIGHT
// =========================

function lineBlocked(x1,y1,x2,y2){

    const dist=Math.hypot(x2-x1,y2-y1);
    const steps=Math.ceil(dist/15);

    for(let i=0;i<=steps;i++){

        const t=i/steps;

        const x=x1+(x2-x1)*t;
        const y=y1+(y2-y1)*t;

        const test={
            x:x-3,
            y:y-3,
            w:6,
            h:6
        };

        for(const w of walls){

            if(rectsOverlap(test,w))
                return true;

        }
    }

    return false;
}


// =========================
// VISION GURU
// =========================

function canSee(enemy){

    const ex=enemy.x+enemy.w/2;
    const ey=enemy.y+enemy.h/2;

    const px=player.x+player.w/2;
    const py=player.y+player.h/2;

    const dx=px-ex;
    const dy=py-ey;

    const distance=Math.hypot(dx,dy);

    if(distance>230)
        return false;

    const targetAngle=Math.atan2(dy,dx);

    let diff=targetAngle-enemy.angle;

    while(diff>Math.PI)
        diff-=Math.PI*2;

    while(diff<-Math.PI)
        diff+=Math.PI*2;

    if(Math.abs(diff)>0.7)
        return false;

    return !lineBlocked(ex,ey,px,py);
}


// =========================
// GERAK GURU
// =========================

function moveEnemy(enemy,dx,dy){

    const nx={
        x:enemy.x+dx,
        y:enemy.y,
        w:enemy.w,
        h:enemy.h
    };

    if(!blocked(nx))
        enemy.x+=dx;

    const ny={
        x:enemy.x,
        y:enemy.y+dy,
        w:enemy.w,
        h:enemy.h
    };

    if(!blocked(ny))
        enemy.y+=dy;
}

function chaseEnemy(enemy){

    const tx=player.x-enemy.x;
    const ty=player.y-enemy.y;

    const dist=Math.hypot(tx,ty);

    if(dist<1)return;

    const speed=1.9;

    const dx=tx/dist*speed;
    const dy=ty/dist*speed;

    enemy.angle=Math.atan2(dy,dx);

    const oldX=enemy.x;
    const oldY=enemy.y;

    moveEnemy(enemy,dx,dy);

    if(
        Math.abs(enemy.x-oldX)<.01 &&
        Math.abs(enemy.y-oldY)<.01
    ){

        moveEnemy(enemy,dx,0);
        moveEnemy(enemy,0,dy);

    }
}


// =========================
// UPDATE GURU
// =========================

function updateEnemies(){

    for(const e of enemies){

        if(e.state==="patrol"){

            e.x+=e.vx;
            e.angle=e.vx>0?0:Math.PI;

            if(e.x<e.minX){

                e.x=e.minX;
                e.vx=Math.abs(e.vx);

            }

            if(e.x>e.maxX){

                e.x=e.maxX;
                e.vx=-Math.abs(e.vx);

            }

            if(canSee(e)){

                e.state="chase";
                e.lastSeenX=player.x;
                e.lastSeenY=player.y;

                showMessage(
                    "🚨 GURU MELIHATMU! LARI!",
                    100
                );

            }
        }

        else if(e.state==="chase"){

            if(canSee(e)){

                e.lastSeenX=player.x;
                e.lastSeenY=player.y;

            }
            else{

                e.state="search";
                e.searchTimer=180;

                showMessage(
                    "👀 Guru kehilanganmu...",
                    100
                );

            }

            chaseEnemy(e);
        }

        else if(e.state==="search"){

            const dx=e.lastSeenX-e.x;
            const dy=e.lastSeenY-e.y;

            const dist=Math.hypot(dx,dy);

            if(dist>8){

                const speed=1.3;

                moveEnemy(
                    e,
                    dx/dist*speed,
                    dy/dist*speed
                );

                e.angle=Math.atan2(dy,dx);

            }
            else{

                e.searchTimer--;

            }

            if(canSee(e)){

                e.state="chase";

                showMessage(
                    "🚨 GURU MENEMUKANMU!",
                    100
                );

            }

            if(e.searchTimer<=0){

                e.state="patrol";

                showMessage(
                    "😮‍💨 Guru kembali patroli.",
                    80
                );

            }
        }

        if(rectsOverlap(player,e))
            damage();
    }
}


// =========================
// SATPAM AI
// =========================

function canGuardSee(){

    const ex=guard.x+guard.w/2;
    const ey=guard.y+guard.h/2;

    const px=player.x+player.w/2;
    const py=player.y+player.h/2;

    const dx=px-ex;
    const dy=py-ey;

    const distance=Math.hypot(dx,dy);

    if(distance>300)
        return false;

    const targetAngle=Math.atan2(dy,dx);

    let diff=targetAngle-guard.angle;

    while(diff>Math.PI)
        diff-=Math.PI*2;

    while(diff<-Math.PI)
        diff+=Math.PI*2;

    if(Math.abs(diff)>0.8)
        return false;

    return !lineBlocked(ex,ey,px,py);
}

function moveGuard(dx,dy){

    const nx={
        x:guard.x+dx,
        y:guard.y,
        w:guard.w,
        h:guard.h
    };

    if(!blocked(nx))
        guard.x+=dx;

    const ny={
        x:guard.x,
        y:guard.y+dy,
        w:guard.w,
        h:guard.h
    };

    if(!blocked(ny))
        guard.y+=dy;
}

function chaseGuard(){

    const tx=player.x-guard.x;
    const ty=player.y-guard.y;

    const dist=Math.hypot(tx,ty);

    if(dist<1)return;

    const speed=2;

    const dx=tx/dist*speed;
    const dy=ty/dist*speed;

    guard.angle=Math.atan2(dy,dx);

    const oldX=guard.x;
    const oldY=guard.y;

    moveGuard(dx,dy);

    if(
        Math.abs(guard.x-oldX)<.01 &&
        Math.abs(guard.y-oldY)<.01
    ){

        moveGuard(dx,0);
        moveGuard(0,dy);

    }
}

function updateGuard(){

    if(guard.state==="patrol"){

        guard.x+=guard.vx;
        guard.angle=guard.vx>0?0:Math.PI;

        if(guard.x<guard.minX){

            guard.x=guard.minX;
            guard.vx=Math.abs(guard.vx);

        }

        if(guard.x>guard.maxX){

            guard.x=guard.maxX;
            guard.vx=-Math.abs(guard.vx);

        }

        if(canGuardSee()){

            guard.state="chase";

            guard.lastSeenX=player.x;
            guard.lastSeenY=player.y;

            showMessage(
                "🚨 SATPAM MELIHATMU! CEPAT LARI!",
                120
            );

        }
    }

    else if(guard.state==="chase"){

        if(canGuardSee()){

            guard.lastSeenX=player.x;
            guard.lastSeenY=player.y;

        }
        else{

            guard.state="search";
            guard.searchTimer=220;

            showMessage(
                "👀 Satpam kehilanganmu...",
                100
            );

        }

        chaseGuard();
    }

    else if(guard.state==="search"){

        const dx=guard.lastSeenX-guard.x;
        const dy=guard.lastSeenY-guard.y;

        const dist=Math.hypot(dx,dy);

        if(dist>8){

            const speed=1.4;

            moveGuard(
                dx/dist*speed,
                dy/dist*speed
            );

            guard.angle=Math.atan2(dy,dx);

        }
        else{

            guard.searchTimer--;

        }

        if(canGuardSee()){

            guard.state="chase";

            showMessage(
                "🚨 SATPAM MENEMUKANMU!",
                100
            );

        }

        if(guard.searchTimer<=0){

            guard.state="patrol";

            showMessage(
                "😮‍💨 Satpam kembali berjaga.",
                80
            );

        }
    }

    if(rectsOverlap(player,guard))
        damage();
}


// =========================
// DAMAGE
// =========================

function damage(){

    if(damageCooldown>0)
        return;

    damageCooldown=100;

    lives--;

    document.getElementById("lives")
        .textContent=lives;

    player.x=230;
    player.y=850;

    for(const e of enemies){

        e.state="patrol";
        e.searchTimer=0;

    }

    guard.state="patrol";
    guard.searchTimer=0;

    showMessage(
        "😭 Ketahuan! Cari jalan lain!",
        120
    );

    if(lives<=0){

        gameRunning=false;

        document.getElementById("gameover")
            .classList.remove("hidden");

    }
}


// =========================
// UPDATE KOIN
// =========================

function updateCoins(){

    for(const c of coins){

        if(c.taken)
            continue;

        const dist=Math.hypot(
            player.x+player.w/2-c.x,
            player.y+player.h/2-c.y
        );

        if(dist<45){

            c.taken=true;
            score+=100;

            document.getElementById("score")
                .textContent=score;

            showMessage(
                "⭐ Koin +100!",
                60
            );
        }
    }
}


// =========================
// UPDATE KUNCI
// =========================

function updateKey(){

    if(key.collected)
        return;

    const dist=Math.hypot(
        player.x+player.w/2-key.x,
        player.y+player.h/2-key.y
    );

    if(dist<50){

        key.collected=true;

        document.getElementById("keyText")
            .textContent="SUDAH";

        showMessage(
            "🔑 Kunci didapat! Sekarang menuju gerbang!",
            150
        );
    }
}


// =========================
// UPDATE GERBANG
// =========================

function updateGate(){

    if(key.collected)
        gate.open=true;
}


// =========================
// CEK MENANG
// =========================

function checkWin(){

    if(!key.collected)
        return;

    if(!gate.open)
        return;

    if(rectsOverlap(player,exitZone)){

        gameWon=true;
        gameRunning=false;

        document.getElementById("finalScore")
            .textContent=score;

        document.getElementById("win")
            .classList.remove("hidden");
    }
}


// =========================
// GAMBAR LANTAI
// =========================

function drawFloor(){

    ctx.fillStyle="#d8c79e";
    ctx.fillRect(0,0,WORLD_W,WORLD_H);

    const size=50;

    for(let y=0;y<WORLD_H;y+=size){

        for(let x=0;x<WORLD_W;x+=size){

            ctx.strokeStyle="rgba(90,70,45,.10)";

            ctx.strokeRect(
                x-cameraX,
                y-cameraY,
                size,
                size
            );
        }
    }
}


// =========================
// GAMBAR DINDING
// =========================

function drawWalls(){

    for(const w of walls){

        const x=w.x-cameraX;
        const y=w.y-cameraY;

        ctx.fillStyle="#eee4ca";
        ctx.fillRect(x,y,w.w,w.h);

        ctx.fillStyle="#b33b3b";

        if(w.h>100){

            ctx.fillRect(
                x,
                y+w.h-18,
                w.w,
                18
            );

        }
        else{

            ctx.fillRect(
                x,
                y,
                w.w,
                Math.min(12,w.h)
            );

        }

        ctx.strokeStyle="#8d7c62";
        ctx.strokeRect(x,y,w.w,w.h);
    }
}


// =========================
// GAMBAR FURNITURE
// =========================

function drawFurniture(){

    for(const f of furniture){

        const x=f.x-cameraX;
        const y=f.y-cameraY;

        if(f.type==="desk"){

            ctx.fillStyle="#9b633c";
            ctx.fillRect(x,y,65,38);

            ctx.fillStyle="#6f4328";

            ctx.fillRect(x+8,y+35,7,25);
            ctx.fillRect(x+50,y+35,7,25);

            ctx.fillStyle="#4b77a8";
            ctx.fillRect(x+18,y+42,30,20);
        }

        if(f.type==="teacherDesk"){

            ctx.fillStyle="#70472d";
            ctx.fillRect(x,y,80,50);

            ctx.fillStyle="#4c3020";

            ctx.fillRect(x+8,y+45,8,25);
            ctx.fillRect(x+64,y+45,8,25);
        }

        if(f.type==="labTable"){

            ctx.fillStyle="#aaa";
            ctx.fillRect(x,y,120,55);

            ctx.fillStyle="#555";

            ctx.fillRect(x+10,y+50,8,25);
            ctx.fillRect(x+100,y+50,8,25);

            ctx.fillStyle="#7bc6d9";
            ctx.fillRect(x+25,y+10,18,25);

            ctx.fillStyle="#d96b6b";
            ctx.fillRect(x+65,y+15,15,20);
        }

        if(f.type==="board"){

            ctx.fillStyle="#24583d";
            ctx.fillRect(x,y,220,70);

            ctx.fillStyle="#fff";
            ctx.font="bold 14px Arial";

            ctx.fillText(
                "PELAJARAN HARI INI",
                x+20,y+28
            );

            ctx.fillText(
                "JANGAN BERISIK!",
                x+20,y+52
            );
        }

        if(f.type==="cabinet"){

            ctx.fillStyle="#8a5b36";
            ctx.fillRect(x,y,50,70);

            ctx.strokeStyle="#d6a56b";
            ctx.strokeRect(x+6,y+8,38,52);
        }

        if(f.type==="bench"){

            ctx.fillStyle="#77482d";
            ctx.fillRect(x,y,110,18);

            ctx.fillStyle="#5c3825";

            ctx.fillRect(x+10,y+15,8,25);
            ctx.fillRect(x+92,y+15,8,25);
        }

        if(f.type==="plant"){

            ctx.fillStyle="#8b5a32";
            ctx.fillRect(x,y+25,30,25);

            ctx.fillStyle="#369447";

            ctx.beginPath();

            ctx.arc(
                x+15,
                y+15,
                22,
                0,
                Math.PI*2
            );

            ctx.fill();
        }

        if(f.type==="trash"){

            ctx.fillStyle="#555";
            ctx.fillRect(x,y,35,45);

            ctx.fillStyle="#222";
            ctx.fillRect(x-3,y-5,41,7);
        }
    }
}


// =========================
// KOIN
// =========================

function drawCoins(){

    for(const c of coins){

        if(c.taken)
            continue;

        const x=c.x-cameraX;
        const y=c.y-cameraY;

        ctx.fillStyle="#ffd43b";

        ctx.beginPath();

        ctx.arc(
            x,y,
            13,
            0,
            Math.PI*2
        );

        ctx.fill();

        ctx.strokeStyle="#a87b00";
        ctx.lineWidth=3;
        ctx.stroke();

        ctx.fillStyle="#fff3a1";
        ctx.font="bold 12px Arial";

        ctx.fillText(
            "★",
            x-6,
            y+5
        );
    }
}


// =========================
// KUNCI
// =========================

function drawKey(){

    if(key.collected)
        return;

    const x=key.x-cameraX;
    const y=key.y-cameraY;

    ctx.save();

    ctx.translate(x,y);

    ctx.rotate(
        Math.sin(Date.now()/250)*.15
    );

    ctx.fillStyle="#ffd43b";

    ctx.beginPath();

    ctx.arc(
        0,0,
        10,
        0,
        Math.PI*2
    );

    ctx.fill();

    ctx.fillRect(
        5,-4,
        32,8
    );

    ctx.fillRect(
        27,4,
        7,12
    );

    ctx.fillRect(
        18,4,
        7,9
    );

    ctx.restore();
}


// =========================
// GERBANG
// =========================

function drawGate(){

    const x=gate.x-cameraX;
    const y=gate.y-cameraY;

    ctx.fillStyle="#b9b0a0";

    ctx.fillRect(
        x,y,
        gate.w,
        gate.h
    );

    // TIANG
    ctx.fillStyle="#777";

    ctx.fillRect(
        x,y-40,
        45,210
    );

    ctx.fillRect(
        x+355,y-40,
        45,210
    );

    // NAMA SEKOLAH
    ctx.fillStyle="#fff";
    ctx.font="bold 25px Arial";
    ctx.textAlign="center";

    ctx.fillText(
        "SMA BAHAGIA",
        x+200,
        y-55
    );

    ctx.textAlign="left";

    if(gate.open){

        ctx.fillStyle="#54a85d";

        ctx.fillRect(
            x+45,y+20,
            150,120
        );

        ctx.fillRect(
            x+205,y+20,
            150,120
        );

        ctx.strokeStyle="#315d36";

        for(let i=0;i<7;i++){

            ctx.strokeRect(
                x+55+i*20,
                y+25,
                8,110
            );

            ctx.strokeRect(
                x+215+i*20,
                y+25,
                8,110
            );
        }

    }
    else{

        ctx.fillStyle="#31549b";

        ctx.fillRect(
            x+45,y+20,
            150,120
        );

        ctx.fillRect(
            x+205,y+20,
            150,120
        );

        ctx.fillStyle="#fff";
        ctx.font="bold 18px Arial";

        ctx.fillText(
            "TERKUNCI",
            x+145,
            y+90
        );
    }
}


// =========================
// VISION
// =========================

function drawVision(enemy,isGuard=false){

    const ex=enemy.x+enemy.w/2-cameraX;
    const ey=enemy.y+enemy.h/2-cameraY;

    const range=isGuard?300:230;
    const width=isGuard?.8:.7;

    ctx.save();

    ctx.globalAlpha=.16;

    ctx.fillStyle=
        enemy.state==="chase"
        ?"red"
        :"yellow";

    ctx.beginPath();

    ctx.moveTo(ex,ey);

    ctx.arc(
        ex,
        ey,
        range,
        enemy.angle-width,
        enemy.angle+width
    );

    ctx.closePath();
    ctx.fill();

    ctx.restore();
}


// =========================
// GURU
// =========================

function drawEnemies(){

    for(const e of enemies){

        drawVision(e,false);

        const x=e.x-cameraX;
        const y=e.y-cameraY;

        ctx.fillStyle=
            e.state==="chase"
            ?"red"
            :"#31549b";

        ctx.fillRect(
            x+7,y+20,
            28,34
        );

        ctx.fillStyle="#f1c7a5";

        ctx.beginPath();

        ctx.arc(
            x+21,y+12,
            12,
            0,
            Math.PI*2
        );

        ctx.fill();

        ctx.fillStyle="#333";

        ctx.beginPath();

        ctx.arc(
            x+21,y+8,
            11,
            Math.PI,
            Math.PI*2
        );

        ctx.fill();

        ctx.fillStyle="#f1c7a5";

        ctx.fillRect(
            x,y+24,
            8,22
        );

        ctx.fillRect(
            x+34,y+24,
            8,22
        );

        ctx.fillStyle="#222";
        ctx.font="bold 11px Arial";

        ctx.fillText(
            "GURU",
            x-2,y-7
        );
    }
}


// =========================
// SATPAM
// =========================

function drawGuard(){

    drawVision(guard,true);

    const x=guard.x-cameraX;
    const y=guard.y-cameraY;

    ctx.fillStyle=
        guard.state==="chase"
        ?"red"
        :"#273c75";

    ctx.fillRect(
        x+7,y+20,
        34,38
    );

    ctx.fillStyle="#d9a47f";

    ctx.beginPath();

    ctx.arc(
        x+24,y+12,
        13,
        0,
        Math.PI*2
    );

    ctx.fill();

    // TOPI
    ctx.fillStyle="#202833";

    ctx.fillRect(
        x+10,y,
        28,8
    );

    ctx.fillRect(
        x+5,y+7,
        38,5
    );

    ctx.fillStyle="#d9a47f";

    ctx.fillRect(
        x,y+26,
        8,22
    );

    ctx.fillRect(
        x+40,y+26,
        8,22
    );

    ctx.fillStyle="#fff";
    ctx.font="bold 11px Arial";

    ctx.fillText(
        "SATPAM",
        x-8,y-12
    );
}


// =========================
// PLAYER
// =========================

function drawPlayer(){

    const x=player.x-cameraX;
    const y=player.y-cameraY;

    if(
        playerImg.complete &&
        playerImg.naturalWidth>0
    ){

        ctx.drawImage(
            playerImg,
            x-5,y-8,
            52,68
        );

    }
    else{

        ctx.fillStyle="#2f6fed";

        ctx.fillRect(
            x+7,y+20,
            28,35
        );

        ctx.fillStyle="#f1c7a5";

        ctx.beginPath();

        ctx.arc(
            x+21,y+12,
            13,
            0,
            Math.PI*2
        );

        ctx.fill();
    }
}


// =========================
// LABEL SEKOLAH
// =========================

function drawLabels(){

    ctx.font="bold 22px Arial";
    ctx.fillStyle="rgba(30,30,30,.55)";

    const labels=[

        [180,180,"KELAS X-A"],
        [910,180,"KELAS X-B"],
        [1590,180,"KELAS X-C"],
        [2310,180,"LAB IPA"],

        [180,1030,"KELAS XI"],
        [960,1030,"KELAS XII"],
        [1730,1030,"TOILET"],
        [2190,1030,"TANGGA"]
    ];

    for(const l of labels){

        ctx.fillText(
            l[2],
            l[0]-cameraX,
            l[1]-cameraY
        );
    }
}


// =========================
// CAMERA
// =========================

function updateCamera(){

    cameraX=
        player.x+
        player.w/2-
        W/2;

    cameraY=
        player.y+
        player.h/2-
        H/2;

    cameraX=Math.max(
        0,
        Math.min(
            cameraX,
            WORLD_W-W
        )
    );

    cameraY=Math.max(
        0,
        Math.min(
            cameraY,
            WORLD_H-H
        )
    );
}


// =========================
// DRAW
// =========================

function draw(){

    ctx.clearRect(0,0,W,H);

    drawFloor();
    drawWalls();
    drawLabels();
    drawFurniture();

    drawCoins();
    drawKey();

    drawGate();

    drawEnemies();
    drawGuard();

    drawPlayer();

    if(key.collected){

        const x=exitZone.x-cameraX;
        const y=exitZone.y-cameraY;

        ctx.strokeStyle="#27b95a";
        ctx.lineWidth=4;

        ctx.strokeRect(
            x,y,
            exitZone.w,
            exitZone.h
        );

        ctx.fillStyle="#27b95a";
        ctx.font="bold 16px Arial";

        ctx.fillText(
            "KELUAR!",
            x+45,
            y+60
        );
    }
}


// =========================
// UPDATE
// =========================

function update(){

    if(!gameRunning)
        return;

    if(damageCooldown>0)
        damageCooldown--;

    let dx=0;
    let dy=0;

    if(keys.left)dx--;
    if(keys.right)dx++;
    if(keys.up)dy--;
    if(keys.down)dy++;

    const length=Math.hypot(dx,dy);

    if(length>0){

        dx/=length;
        dy/=length;

        const speed=
            keys.run
            ?6.0
            :player.speed;

        movePlayer(
            dx*speed,
            dy*speed
        );
    }

    updateEnemies();
    updateGuard();

    updateCoins();
    updateKey();
    updateGate();

    checkWin();
    updateCamera();

    if(messageTimer>0)
        messageTimer--;
}


// =========================
// LOOP
// =========================

function loop(){

    update();
    draw();

    requestAnimationFrame(loop);
}


// =========================
// START
// =========================

function startGame(){

    document.getElementById("menu")
        .classList.add("hidden");

    gameRunning=true;

    showMessage(
        "Cari 🔑 kunci lalu menuju 🚪 gerbang!",
        180
    );
}


// =========================
// RESTART
// =========================

function restartGame(){

    lives=3;
    score=0;

    player.x=230;
    player.y=850;

    randomizeKey();

    gate.open=false;
    gameWon=false;

    for(const c of coins)
        c.taken=false;

    for(const e of enemies){

        e.state="patrol";
        e.searchTimer=0;

    }

    guard.state="patrol";
    guard.searchTimer=0;

    document.getElementById("lives")
        .textContent=lives;

    document.getElementById("score")
        .textContent=score;

    document.getElementById("keyText")
        .textContent="BELUM";

    document.getElementById("gameover")
        .classList.add("hidden");

    document.getElementById("win")
        .classList.add("hidden");

    gameRunning=true;

    showMessage(
        "Mulai lagi! Jangan sampai ketahuan.",
        120
    );
}


// =========================
// KEYBOARD
// =========================

addEventListener("keydown",e=>{

    if(
        e.key==="ArrowUp"||
        e.key.toLowerCase()==="w"
    )
        keys.up=true;

    if(
        e.key==="ArrowDown"||
        e.key.toLowerCase()==="s"
    )
        keys.down=true;

    if(
        e.key==="ArrowLeft"||
        e.key.toLowerCase()==="a"
    )
        keys.left=true;

    if(
        e.key==="ArrowRight"||
        e.key.toLowerCase()==="d"
    )
        keys.right=true;

    if(e.key==="Shift")
        keys.run=true;
});

addEventListener("keyup",e=>{

    if(
        e.key==="ArrowUp"||
        e.key.toLowerCase()==="w"
    )
        keys.up=false;

    if(
        e.key==="ArrowDown"||
        e.key.toLowerCase()==="s"
    )
        keys.down=false;

    if(
        e.key==="ArrowLeft"||
        e.key.toLowerCase()==="a"
    )
        keys.left=false;

    if(
        e.key==="ArrowRight"||
        e.key.toLowerCase()==="d"
    )
        keys.right=false;

    if(e.key==="Shift")
        keys.run=false;
});


// =========================
// TOUCH CONTROL
// =========================

document.querySelectorAll("[data-key]")
.forEach(button=>{

    const keyName=button.dataset.key;

    const start=e=>{

        e.preventDefault();
        keys[keyName]=true;

    };

    const end=e=>{

        e.preventDefault();
        keys[keyName]=false;

    };

    button.addEventListener(
        "touchstart",
        start,
        {passive:false}
    );

    button.addEventListener(
        "touchend",
        end,
        {passive:false}
    );

    button.addEventListener(
        "touchcancel",
        end,
        {passive:false}
    );

    button.addEventListener(
        "mousedown",
        start
    );

    button.addEventListener(
        "mouseup",
        end
    );

    button.addEventListener(
        "mouseleave",
        end
    );
});


// KUNCI RANDOM SAAT LOAD
randomizeKey();

loop();
</script>

</body>
</html>
