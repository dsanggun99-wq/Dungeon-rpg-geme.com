<!DOCTYPE html>
 <html lang="id">
 <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>Dungeon Escape: Pixel RPG</title>
     <style>
         * { margin: 0; padding: 0; box-sizing: border-box; }
         body { background: #1a1a1a; display: flex; flex-direction: column; align-items: center; padding-top: 20px; }
         #gameCanvas { border: 3px solid #444; background: url('https://via.placeholder.com/800x600/3a2f2f/5a4f4f?text=Dungeon+Latar+Pixel') no-repeat center; background-size: cover; }
         #status { color: #fff; font-family: 'Pixelify Sans', sans-serif; margin: 10px 0; font-size: 18px; line-height: 24px; }
         #levelUpPopup { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #2a2a2a; border: 2px solid #ffd700; padding: 20px; color: #fff; font-family: 'Pixelify Sans'; display: none; z-index: 10; }
         #escapePopup { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #2a2a2a; border: 2px solid #00ff00; padding: 30px; color: #fff; font-family: 'Pixelify Sans'; font-size: 24px; display: none; z-index: 20; }
     </style>
     <!-- Font Pixel -->
     <link href="https://fonts.googleapis.com/css2?family=Pixelify+Sans:wght@400;700&display=swap" rel="stylesheet">
 </head>
 <body>
     <div id="status">Level: 1 | XP: 0/100 | HP: 100/100 | Serangan: 10 | Kecepatan: 5 | Item: 0/5 | Potion: 0 | Dekat NPC: Tidak | Pintu Keluar: Tidak Ada</div>
     <div id="levelUpPopup">LEVEL UP! 🎉<br>HP +30 | Serangan +5 | Kecepatan +1</div>
     <div id="escapePopup">KAMU MELARIKAN DIRI! ✨<br>Selamat, kamu selesai game!</div>
     <canvas id="gameCanvas" width="800" height="600"></canvas>
     <!-- Efek Suara -->
     <audio id="walkSound" preload="auto">
         <source src="https://www.soundjay.com/footsteps/sounds/footsteps-01.mp3" type="audio/mpeg">
     
     <audio id="attackSound" preload="auto">
         <source src="https://www.soundjay.com/weapons/sounds/sword-1.mp3" type="audio/mpeg">
     
     <audio id="pickupSound" preload="auto">
         <source src="https://www.soundjay.com/button/sounds/button-07.mp3" type="audio/mpeg">
     
     <audio id="levelUpSound" preload="auto">
         <source src="https://www.soundjay.com/misc/sounds/level-up-01.mp3" type="audio/mpeg">
     
     <audio id="healSound" preload="auto">
         <source src="https://www.soundjay.com/misc/sounds/heal-01.mp3" type="audio/mpeg">
     
     <audio id="escapeSound" preload="auto">
         <source src="https://www.soundjay.com/win/sounds/win-01.mp3" type="audio/mpeg">
     
     <script>
         // Inisialisasi Canvas
         const canvas = document.getElementById('gameCanvas');
         const ctx = canvas.getContext('2d');
         const statusDiv = document.getElementById('status');
         const levelUpPopup = document.getElementById('levelUpPopup');
         const escapePopup = document.getElementById('escapePopup');
         // Suara
         const walkSound = document.getElementById('walkSound');
         const attackSound = document.getElementById('attackSound');
         const pickupSound = document.getElementById('pickupSound');
         const levelUpSound = document.getElementById('levelUpSound');
         const healSound = document.getElementById('healSound');
         const escapeSound = document.getElementById('escapeSound');
         walkSound.volume = 0.3;
         levelUpSound.volume = 0.5;
         escapeSound.volume = 0.7;
         // Karakter Laki-Laki
         const player = {
             x: 400, y: 300, size: 32,
             level: 1, xp: 0, xpNeeded: 100,
             maxHp: 100, hp: 100,
             attack: 10, speed: 5,
             items: 0, potions: 0, // Tambahin potion
             direction: 'down',
             frames: {
                 down: [[255,255,0], [255,200,0]],
                 up: [[0,255,255], [0,200,255]],
                 left: [[255,0,255], [200,0,255]],
                 right: [[0,255,0], [0,200,0]]
             },
             frameIndex: 0,
             frameTimer: 0,
             frameDelay: 10
         };
         // NPC
         const npc = {
             x: 600, y: 400, size: 32,
             color: [150, 75, 0],
             text: "Hai! Kumpulkan item, gunakan potion untuk heal, dan naiki level sampai 10 — pintu keluar akan muncul!"
         };
         // Item Utama (5 buah)
         const mainItems = Array.from({length: 5}, () => ({
             x: Math.random() * (canvas.width - 20) + 10,
             y: Math.random() * (canvas.height - 20) + 10,
             size: 20,
             color: [Math.random()*255, Math.random()*255, Math.random()*255],
             collected: false,
             xpGive: 30
         }));
         // Item Potion Heal (3 buah, posisi acak)
         const potions = Array.from({length: 3}, () => ({
             x: Math.random() * (canvas.width - 20) + 10,
             y: Math.random() * (canvas.height - 20) + 10,
             size: 18,
             color: [0, 255, 100], // Warna hijau muda (mudah dikenali)
             collected: false,
             healAmount: 50 // Heal 50 HP per potion
         }));
         // Musuh (4 musuh buat lebih banyak XP)
         const enemies = [
             { x: 100, y: 100, size: 32, color: [255,0,0], hp: 50, alive: true, xpGive: 50 },
             { x: 700, y: 500, size: 32, color: [255,100,100], hp: 70, alive: true, xpGive: 70 },
             { x: 200, y: 500, size: 32, color: [200,0,200], hp: 90, alive: true, xpGive: 90 },
             { x: 600, y: 100, size: 32, color: [0,0,255], hp: 110, alive: true, xpGive: 110 }
         ];
         // Pintu Keluar (muncul kalo level 10)
         const exitDoor = {
             x: 400, y: 100, // Posisi tengah atas
             size: 40,
             color: [255, 165, 0], // Warna oranye (menarik perhatian)
             isVisible: false
         };
         // Input Keyboard
         const keys = { up: false, down: false, left: false, right: false, attack: false, usePotion: false };
         document.addEventListener('keydown', (e) => {
             switch(e.key) {
                 case 'w': keys.up = true; break;
                 case 's': keys.down = true; break;
                 case 'a': keys.left = true; break;
                 case 'd': keys.right = true; break;
                 case ' ': keys.attack = true; break;
                 case 'h': keys.usePotion = true; break; // Tekan H untuk pake potion
             }
         });
         document.addEventListener('keyup', (e) => {
             switch(e.key) {
                 case 'w': keys.up = false; break;
                 case 's': keys.down = false; break;
                 case 'a': keys.left = false; break;
                 case 'd': keys.right = false; break;
                 case ' ': keys.attack = false; break;
                 case 'h': keys.usePotion = false; break;
             }
         });
         // Fungsi Cek Jarak
         function checkDistance(obj1, obj2) {
             return Math.sqrt(Math.pow(obj1.x - obj2.x, 2) + Math.pow(obj1.y - obj2.y, 2));
         }
         // Fungsi Gambar Pixel
         function drawPixel(x, y, size, color) {
             ctx.fillStyle = `rgb(${color[0]},${color[1]},${color[2]})`;
             ctx.fillRect(x, y, size, size);
         }
         // Fungsi Level Up
         function levelUp() {
             player.level++;
             player.xp -= player.xpNeeded;
             player.xpNeeded = Math.floor(player.xpNeeded * 1.5);
             player.maxHp += 30;
             player.hp = player.maxHp;
             player.attack += 5;
             player.speed += 1;
             levelUpSound.play();
             levelUpPopup.style.display = 'block';
             setTimeout(() => levelUpPopup.style.display = 'none', 2000);
             // Tampilkan pintu keluar kalo level 10
             if (player.level === 10) {
                 exitDoor.isVisible = true;
             }
         }
         // Loop Game
         function gameLoop() {
             ctx.clearRect(0, 0, canvas.width, canvas.height);
             // Update Gerakan
             let moving = false;
             if (keys.up) { player.y -= player.speed; player.direction = 'up'; moving = true; }
             if (keys.down) { player.y += player.speed; player.direction = 'down'; moving = true; }
             if (keys.left) { player.x -= player.speed; player.direction = 'left'; moving = true; }
             if (keys.right) { player.x += player.speed; player.direction = 'right'; moving = true; }
             // Batas Dungeon
             player.x = Math.max(player.size/2, Math.min(canvas.width - player.size/2, player.x));
             player.y = Math.max(player.size/2, Math.min(canvas.height - player.size/2, player.y));
             // Animasi + Suara Langkah
             if (moving) {
                 player.frameTimer++;
                 if (player.frameTimer >= player.frameDelay) {
                     player.frameIndex = (player.frameIndex + 1) % 2;
                     player.frameTimer = 0;
                     if (!walkSound.playing) walkSound.play();
                 }
             } else {
                 player.frameIndex = 0;
                 walkSound.pause();
                 walkSound.currentTime = 0;
             }
             // Gunakan Potion (tekan H)
             if (keys.usePotion && player.potions > 0 && player.hp < player.maxHp) {
                 player.hp = Math.min(player.maxHp, player.hp + potions[0].healAmount);
                 player.potions--;
                 healSound.play();
                 keys.usePotion = false;
             }
             // Efek Serangan
             if (keys.attack) {
                 attackSound.play();
                 const closestEnemy = enemies.find(e => e.alive && checkDistance(player, e) < 100);
                 if (closestEnemy) {
                     ctx.strokeStyle = '#ff0';
                     ctx.lineWidth = 3;
                     ctx.beginPath();
                     ctx.moveTo(player.x, player.y);
                     ctx.lineTo(closestEnemy.x, closestEnemy.y);
                     ctx.stroke();
                     closestEnemy.hp -= player.attack;
                     if (closestEnemy.hp <= 0) {
                         closestEnemy.alive = false;
                         player.xp += closestEnemy.xpGive;
                     }
                 }
                 keys.attack = false;
             }
             // Kumpulkan Item Utama
             mainItems.forEach(item => {
                 if (!item.collected && checkDistance(player, item) < 30) {
                     item.collected = true;
                     player.items++;
                     player.xp += item.xpGive;
                     pickupSound.play();
                 }
             });
             // Kumpulkan Potion
             potions.forEach(potion => {
                 if (!potion.collected && checkDistance(player, potion) < 25) {
                     potion.collected = true;
                     player.potions++;
                     pickupSound.play();
                 }
             });
             // Cek Level Up
             while (player.xp >= player.xpNeeded) {
                 levelUp();
             }
             // Cek Keluar Dungeon (masuk ke pintu)
             if (exitDoor.isVisible && checkDistance(player, exitDoor) < 40) {
                 escapeSound.play();
                 escapePopup.style.display = 'block';
                 cancelAnimationFrame(gameLoop); // Hentikan game
             }
             // Update Status
             const nearNpc = checkDistance(player, npc) < 40 ? "Ya (Tekan G untuk bicara)" : "Tidak";
             const exitStatus = exitDoor.isVisible ? "Ada (di Tengah Atas)" : "Tidak Ada";
             statusDiv.textContent = `Level: ${player.level} | XP: ${player.xp}/${player.xpNeeded} | HP: ${player.hp}/${player.maxHp} | Serangan: ${player.attack} | Kecepatan: ${player.speed} | Item: ${player.items}/5 | Potion: ${player.potions} | Dekat NPC: ${nearNpc} | Pintu Keluar: ${exitStatus}`;
             // Bicara dengan NPC
             document.addEventListener('keydown', (e) => {
                 if (e.key === 'g' && checkDistance(player, npc) < 40) {
                     alert(`NPC: ${npc.text}`);
                 }
             }, { once: true });
             // Gambar Semua Elemen
             mainItems.forEach(item => { if (!item.collected) drawPixel(item.x, item.y, item.size, item.color); });
             potions.forEach(potion => { if (!potion.collected) drawPixel(potion.x, potion.y, potion.size, potion.color); });
             drawPixel(npc.x, npc.y, npc.size, npc.color);
             enemies.forEach(enemy => { if (enemy.alive) drawPixel(enemy.x, enemy.y, enemy.size, enemy.color); });
             if (exitDoor.isVisible) drawPixel(exitDoor.x - exitDoor.size/2, exitDoor.y - exitDoor.size/2, exitDoor.size, exitDoor.color); // Pintu di tengah posisi
             const currentFrame = player.frames[player.direction][player.frameIndex];
             drawPixel(player.x - player.size/2, player.y - player.size/2, player.size, currentFrame);
            <!DOCTYPE html>
 <html lang="id">
 <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>Dungeon Escape: Pixel RPG</title>
     <style>
         * { margin: 0; padding: 0; box-sizing: border-box; }
         body { background: #1a1a1a; display: flex; flex-direction: column; align-items: center; padding-top: 20px; }
         #gameCanvas { border: 3px solid #444; background: url('https://via.placeholder.com/800x600/3a2f2f/5a4f4f?text=Dungeon+Latar+Pixel') no-repeat center; background-size: cover; }
         #status { color: #fff; font-family: 'Pixelify Sans', sans-serif; margin: 10px 0; font-size: 18px; line-height: 24px; }
         #levelUpPopup { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #2a2a2a; border: 2px solid #ffd700; padding: 20px; color: #fff; font-family: 'Pixelify Sans'; display: none; z-index: 10; }
         #escapePopup { position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); background: #2a2a2a; border: 2px solid #00ff00; padding: 30px; color: #fff; font-family: 'Pixelify Sans'; font-size: 24px; display: none; z-index: 20; }
     </style>
     <!-- Font Pixel -->
     <link href="https://fonts.googleapis.com/css2?family=Pixelify+Sans:wght@400;700&display=swap" rel="stylesheet">
 </head>
 <body>
     <div id="status">Level: 1 | XP: 0/100 | HP: 100/100 | Serangan: 10 | Kecepatan: 5 | Item: 0/5 | Potion: 0 | Dekat NPC: Tidak | Pintu Keluar: Tidak Ada</div>
     <div id="levelUpPopup">LEVEL UP! 🎉<br>HP +30 | Serangan +5 | Kecepatan +1</div>
     <div id="escapePopup">KAMU MELARIKAN DIRI! ✨<br>Selamat, kamu selesai game!</div>
     <canvas id="gameCanvas" width="800" height="600"></canvas>
     <!-- Efek Suara -->
     <audio id="walkSound" preload="auto">
         <source src="https://www.soundjay.com/footsteps/sounds/footsteps-01.mp3" type="audio/mpeg">
     
     <audio id="attackSound" preload="auto">
         <source src="https://www.soundjay.com/weapons/sounds/sword-1.mp3" type="audio/mpeg">
     
     <audio id="pickupSound" preload="auto">
         <source src="https://www.soundjay.com/button/sounds/button-07.mp3" type="audio/mpeg">
     
     <audio id="levelUpSound" preload="auto">
         <source src="https://www.soundjay.com/misc/sounds/level-up-01.mp3" type="audio/mpeg">
     
     <audio id="healSound" preload="auto">
         <source src="https://www.soundjay.com/misc/sounds/heal-01.mp3" type="audio/mpeg">
     
     <audio id="escapeSound" preload="auto">
         <source src="https://www.soundjay.com/win/sounds/win-01.mp3" type="audio/mpeg">
     
     <script>
         // Inisialisasi Canvas
         const canvas = document.getElementById('gameCanvas');
         const ctx = canvas.getContext('2d');
         const statusDiv = document.getElementById('status');
         const levelUpPopup = document.getElementById('levelUpPopup');
         const escapePopup = document.getElementById('escapePopup');
         // Suara
         const walkSound = document.getElementById('walkSound');
         const attackSound = document.getElementById('attackSound');
         const pickupSound = document.getElementById('pickupSound');
         const levelUpSound = document.getElementById('levelUpSound');
         const healSound = document.getElementById('healSound');
         const escapeSound = document.getElementById('escapeSound');
         walkSound.volume = 0.3;
         levelUpSound.volume = 0.5;
         escapeSound.volume = 0.7;
         // Karakter Laki-Laki
         const player = {
             x: 400, y: 300, size: 32,
             level: 1, xp: 0, xpNeeded: 100,
             maxHp: 100, hp: 100,
             attack: 10, speed: 5,
             items: 0, potions: 0, // Tambahin potion
             direction: 'down',
             frames: {
                 down: [[255,255,0], [255,200,0]],
                 up: [[0,255,255], [0,200,255]],
                 left: [[255,0,255], [200,0,255]],
                 right: [[0,255,0], [0,200,0]]
             },
             frameIndex: 0,
             frameTimer: 0,
             frameDelay: 10
         };
         // NPC
         const npc = {
             x: 600, y: 400, size: 32,
             color: [150, 75, 0],
             text: "Hai! Kumpulkan item, gunakan potion untuk heal, dan naiki level sampai 10 — pintu keluar akan muncul!"
         };
         // Item Utama (5 buah)
         const mainItems = Array.from({length: 5}, () => ({
             x: Math.random() * (canvas.width - 20) + 10,
             y: Math.random() * (canvas.height - 20) + 10,
             size: 20,
             color: [Math.random()*255, Math.random()*255, Math.random()*255],
             collected: false,
             xpGive: 30
         }));
         // Item Potion Heal (3 buah, posisi acak)
         const potions = Array.from({length: 3}, () => ({
             x: Math.random() * (canvas.width - 20) + 10,
             y: Math.random() * (canvas.height - 20) + 10,
             size: 18,
             color: [0, 255, 100], // Warna hijau muda (mudah dikenali)
             collected: false,
             healAmount: 50 // Heal 50 HP per potion
         }));
         // Musuh (4 musuh buat lebih banyak XP)
         const enemies = [
             { x: 100, y: 100, size: 32, color: [255,0,0], hp: 50, alive: true, xpGive: 50 },
             { x: 700, y: 500, size: 32, color: [255,100,100], hp: 70, alive: true, xpGive: 70 },
             { x: 200, y: 500, size: 32, color: [200,0,200], hp: 90, alive: true, xpGive: 90 },
             { x: 600, y: 100, size: 32, color: [0,0,255], hp: 110, alive: true, xpGive: 110 }
         ];
         // Pintu Keluar (muncul kalo level 10)
         const exitDoor = {
             x: 400, y: 100, // Posisi tengah atas
             size: 40,
             color: [255, 165, 0], // Warna oranye (menarik perhatian)
             isVisible: false
         };
         // Input Keyboard
         const keys = { up: false, down: false, left: false, right: false, attack: false, usePotion: false };
         document.addEventListener('keydown', (e) => {
             switch(e.key) {
                 case 'w': keys.up = true; break;
                 case 's': keys.down = true; break;
                 case 'a': keys.left = true; break;
                 case 'd': keys.right = true; break;
                 case ' ': keys.attack = true; break;
                 case 'h': keys.usePotion = true; break; // Tekan H untuk pake potion
             }
         });
         document.addEventListener('keyup', (e) => {
             switch(e.key) {
                 case 'w': keys.up = false; break;
                 case 's': keys.down = false; break;
                 case 'a': keys.left = false; break;
                 case 'd': keys.right = false; break;
                 case ' ': keys.attack = false; break;
                 case 'h': keys.usePotion = false; break;
             }
         });
         // Fungsi Cek Jarak
         function checkDistance(obj1, obj2) {
             return Math.sqrt(Math.pow(obj1.x - obj2.x, 2) + Math.pow(obj1.y - obj2.y, 2));
         }
         // Fungsi Gambar Pixel
         function drawPixel(x, y, size, color) {
             ctx.fillStyle = `rgb(${color[0]},${color[1]},${color[2]})`;
             ctx.fillRect(x, y, size, size);
         }
         // Fungsi Level Up
         function levelUp() {
             player.level++;
             player.xp -= player.xpNeeded;
             player.xpNeeded = Math.floor(player.xpNeeded * 1.5);
             player.maxHp += 30;
             player.hp = player.maxHp;
             player.attack += 5;
             player.speed += 1;
             levelUpSound.play();
             levelUpPopup.style.display = 'block';
             setTimeout(() => levelUpPopup.style.display = 'none', 2000);
             // Tampilkan pintu keluar kalo level 10
             if (player.level === 10) {
                 exitDoor.isVisible = true;
             }
         }
         // Loop Game
         function gameLoop() {
             ctx.clearRect(0, 0, canvas.width, canvas.height);
             // Update Gerakan
             let moving = false;
             if (keys.up) { player.y -= player.speed; player.direction = 'up'; moving = true; }
             if (keys.down) { player.y += player.speed; player.direction = 'down'; moving = true; }
             if (keys.left) { player.x -= player.speed; player.direction = 'left'; moving = true; }
             if (keys.right) { player.x += player.speed; player.direction = 'right'; moving = true; }
             // Batas Dungeon
             player.x = Math.max(player.size/2, Math.min(canvas.width - player.size/2, player.x));
             player.y = Math.max(player.size/2, Math.min(canvas.height - player.size/2, player.y));
             // Animasi + Suara Langkah
             if (moving) {
                 player.frameTimer++;
                 if (player.frameTimer >= player.frameDelay) {
                     player.frameIndex = (player.frameIndex + 1) % 2;
                     player.frameTimer = 0;
                     if (!walkSound.playing) walkSound.play();
                 }
             } else {
                 player.frameIndex = 0;
                 walkSound.pause();
                 walkSound.currentTime = 0;
             }
             // Gunakan Potion (tekan H)
             if (keys.usePotion && player.potions > 0 && player.hp < player.maxHp) {
                 player.hp = Math.min(player.maxHp, player.hp + potions[0].healAmount);
                 player.potions--;
                 healSound.play();
                 keys.usePotion = false;
             }
             // Efek Serangan
             if (keys.attack) {
                 attackSound.play();
                 const closestEnemy = enemies.find(e => e.alive && checkDistance(player, e) < 100);
                 if (closestEnemy) {
                     ctx.strokeStyle = '#ff0';
                     ctx.lineWidth = 3;
                     ctx.beginPath();
                     ctx.moveTo(player.x, player.y);
                     ctx.lineTo(closestEnemy.x, closestEnemy.y);
                     ctx.stroke();
                     closestEnemy.hp -= player.attack;
                     if (closestEnemy.hp <= 0) {
                         closestEnemy.alive = false;
                         player.xp += closestEnemy.xpGive;
                     }
                 }
                 keys.attack = false;
             }
             // Kumpulkan Item Utama
             mainItems.forEach(item => {
                 if (!item.collected && checkDistance(player, item) < 30) {
                     item.collected = true;
                     player.items++;
                     player.xp += item.xpGive;
                     pickupSound.play();
                 }
             });
             // Kumpulkan Potion
             potions.forEach(potion => {
                 if (!potion.collected && checkDistance(player, potion) < 25) {
                     potion.collected = true;
                     player.potions++;
                     pickupSound.play();
                 }
             });
             // Cek Level Up
             while (player.xp >= player.xpNeeded) {
                 levelUp();
             }
             // Cek Keluar Dungeon (masuk ke pintu)
             if (exitDoor.isVisible && checkDistance(player, exitDoor) < 40) {
                 escapeSound.play();
                 escapePopup.style.display = 'block';
                 cancelAnimationFrame(gameLoop); // Hentikan game
             }
             // Update Status
             const nearNpc = checkDistance(player, npc) < 40 ? "Ya (Tekan G untuk bicara)" : "Tidak";
             const exitStatus = exitDoor.isVisible ? "Ada (di Tengah Atas)" : "Tidak Ada";
             statusDiv.textContent = `Level: ${player.level} | XP: ${player.xp}/${player.xpNeeded} | HP: ${player.hp}/${player.maxHp} | Serangan: ${player.attack} | Kecepatan: ${player.speed} | Item: ${player.items}/5 | Potion: ${player.potions} | Dekat NPC: ${nearNpc} | Pintu Keluar: ${exitStatus}`;
             // Bicara dengan NPC
             document.addEventListener('keydown', (e) => {
                 if (e.key === 'g' && checkDistance(player, npc) < 40) {
                     alert(`NPC: ${npc.text}`);
                 }
             }, { once: true });
             // Gambar Semua Elemen
             mainItems.forEach(item => { if (!item.collected) drawPixel(item.x, item.y, item.size, item.color); });
             potions.forEach(potion => { if (!potion.collected) drawPixel(potion.x, potion.y, potion.size, potion.color); });
             drawPixel(npc.x, npc.y, npc.size, npc.color);
             enemies.forEach(enemy => { if (enemy.alive) drawPixel(enemy.x, enemy.y, enemy.size, enemy.color); });
             if (exitDoor.isVisible) drawPixel(exitDoor.x - exitDoor.size/2, exitDoor.y - exitDoor.size/2, exitDoor.size, exitDoor.color); // Pintu di tengah posisi
             const currentFrame = player.frames[player.direction][player.frameIndex];
             drawPixel(player.x - player.size/2, player.y - player.size/2, player.size, currentFrame);
             requestAnimationFrame(gameLoop);
         }
         // Mulai Game
         gameLoop();
     </script>
 (gameLoop);
         }
         // Mulai Game
         gameLoop();
     </script>
 </body>
 </html>
