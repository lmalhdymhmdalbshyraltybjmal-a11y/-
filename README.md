# <!doctype html>
<html lang="ar">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>قاتل الشوارع — بقوة خارقة</title>
  <style>
    :root{
      --bg:#0b1020;
      --ground:#2b2f3a;
      --player:#f4d35e;
      --enemy:#ef476f;
      --text:#eaeaea;
      --accent:#06d6a0;
    }
    html,body{height:100%;margin:0;font-family:system-ui,Segoe UI,Roboto,"Helvetica Neue",Arial}
    body{background:linear-gradient(180deg,var(--bg),#071024);color:var(--text);display:flex;align-items:center;justify-content:center}
    #gameWrap{width:900px;max-width:96vw;background:#071126;border-radius:12px;padding:12px;box-shadow:0 10px 40px rgba(0,0,0,.6)}
    header{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
    header .info{display:flex;gap:12px;align-items:center}
    .meter{background:#122230;border-radius:6px;padding:4px 8px;display:flex;gap:6px;align-items:center}
    .bar{width:160px;height:14px;background:#0b2636;border-radius:6px;overflow:hidden;box-shadow:inset 0 0 6px rgba(0,0,0,.6)}
    .bar > i{display:block;height:100%;width:0;background:linear-gradient(90deg,var(--accent),#24a0ff)}
    button {background:transparent;border:1px solid rgba(255,255,255,.06);color:var(--text);padding:6px 10px;border-radius:8px;cursor:pointer}
    #canvas{background:linear-gradient(#071126,#052034);display:block;border-radius:8px;width:100%;height:520px}
    footer{display:flex;justify-content:space-between;margin-top:8px;gap:8px;flex-wrap:wrap}
    .controls{font-size:13px;opacity:.9}
    .center{display:flex;gap:10px;align-items:center}
    #overlay{position:relative}
    .uiBox{background:rgba(255,255,255,.02);padding:8px;border-radius:8px}
    .small{font-size:13px;opacity:.9}
    /* Mobile touch buttons (optional) */
    .touchRow{display:none}
    @media (max-width:600px){
      #gameWrap{padding:6px}
      .touchRow{display:flex;gap:8px;margin-top:8px;justify-content:center}
      .touchButton{background:rgba(255,255,255,.06);padding:12px;border-radius:10px;font-weight:700;min-width:56px;text-align:center;user-select:none}
    }
  </style>
</head>
<body>
  <div id="gameWrap">
    <header>
      <div class="info">
        <div class="uiBox">
          <strong>قاتل الشوارع</strong>
          <div class="small">نقطة: <span id="score">0</span> — حياة: <span id="hp">100</span></div>
        </div>
        <div class="meter uiBox">
          <div style="font-size:13px">قوة خارقة</div>
          <div class="bar" title="قيمة القوة">
            <i id="superBar"></i>
          </div>
        </div>
      </div>
      <div class="center">
        <button id="btnRestart">إعادة تشغيل</button>
        <div class="small">المفاتيح: ← → للقفز ↑ أو Space — هجوم: Z — قوة خارقة: X</div>
      </div>
    </header>

    <div id="overlay">
      <canvas id="canvas" width="880" height="520"></canvas>
    </div>

    <footer>
      <div class="controls uiBox">
        <div><strong>هدف اللعبة</strong> — هزم الأعداء، اجمع النقاط، واملأ الشريط لاستخدام القوة الخارقة.</div>
      </div>
      <div class="small uiBox">تصميم مبسط: في حال احتجت ميزات إضافية (أسلحة، مراحل، صوت) أضفها واطلب.</div>
    </footer>

    <!-- Touch controls for phones -->
    <div class="touchRow">
      <div class="touchButton" id="tLeft">◀</div>
      <div class="touchButton" id="tJump">⤴</div>
      <div class="touchButton" id="tRight">▶</div>
      <div class="touchButton" id="tHit">ضرب</div>
      <div class="touchButton" id="tSuper">قوة</div>
    </div>
  </div>

  <script>
  // لعبة Canvas — قاتل الشوارع (مبسط)
  (function(){
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');

    // عناصر الواجهة
    const scoreEl = document.getElementById('score');
    const hpEl = document.getElementById('hp');
    const superBarEl = document.getElementById('superBar');
    const btnRestart = document.getElementById('btnRestart');

    const W = canvas.width, H = canvas.height;
    const gravity = 0.9;

    // حالة اللعبة
    let state = {
      running: true,
      score: 0,
      playerHP: 100,
    };

    // اللاعب
    const player = {
      x: 100, y: H - 140, w: 36, h: 48,
      vx: 0, vy: 0,
      speed: 4.2,
      onGround: false,
      facing: 1, // 1 right, -1 left
      attacking: false,
      attackCooldown: 0,
      superPower: 0, // 0..100
      superReady(){ return this.superPower >= 100; }
    };

    // منصات بسيطة (x,y,w,h) — الأرض الرئيسي
    const platforms = [
      {x:0,y:H-80,w:W,h:80},
      {x:320,y:H-180,w:220,h:16},
      {x:650,y:H-260,w:200,h:16}
    ];

    // أعداء
    const enemies = [];
    function spawnEnemy(x,y,type=1){
      enemies.push({
        x, y, w:36, h:44,
        vx: (Math.random()>0.5? -1:1) * (1 + Math.random()*0.6),
        vy:0,
        speed: 1+Math.random()*1.2,
        hp: type===1? 30: 60,
        type,
        alive:true,
        onGround:false,
        flipTimer: 0
      });
    }

    // مؤثر صوتي مرئي مبسط (يمكن إضافة صوت لاحقاً)
    function flash(x,y,r,color){
      // لمزامنة مؤثرات نستخدم قائمة لكن نحافظ على البساطة
      effects.push({x,y,r,color,t:0,d:20});
    }
    const effects = [];

    // زر لمس (mobile)
    const touchMap = {left:false,right:false,jump:false,hit:false,super:false};
    ['tLeft','tRight','tJump','tHit','tSuper'].forEach(id=>{
      const el = document.getElementById(id);
      if(!el) return;
      el.addEventListener('touchstart',e=>{ e.preventDefault(); if(id==='tLeft') touchMap.left=true; if(id==='tRight') touchMap.right=true; if(id==='tJump') touchMap.jump=true; if(id==='tHit') touchMap.hit=true; if(id==='tSuper') touchMap.super=true; });
      el.addEventListener('touchend',e=>{ e.preventDefault(); if(id==='tLeft') touchMap.left=false; if(id==='tRight') touchMap.right=false; if(id==='tJump') touchMap.jump=false; if(id==='tHit') touchMap.hit=false; if(id==='tSuper') touchMap.super=false; });
    });

    // مفاتيح
    const keys = {};
    window.addEventListener('keydown',e=>{ keys[e.key.toLowerCase()]=true; if(['z','x',' '].includes(e.key.toLowerCase())) e.preventDefault(); });
    window.addEventListener('keyup',e=>{ keys[e.key.toLowerCase()]=false; });

    // في البداية نولد بعض الأعداء
    for(let i=0;i<4;i++){
      spawnEnemy(520 + i*120, H - 140, Math.random()>0.7?2:1);
    }

    // تصادم مستطيلات
    function rectsIntersect(a,b){
      return a.x < b.x + b.w && a.x + a.w > b.x && a.y < b.y + b.h && a.y + a.h > b.y;
    }

    // تحديث العالم
    function update(){
      if(!state.running) return;
      // تحكم اللاعب
      const left = keys['arrowleft'] || keys['a'] || touchMap.left;
      const right = keys['arrowright'] || keys['d'] || touchMap.right;
      const jumpPressed = keys['arrowup'] || keys['w'] || keys[' ' ] || touchMap.jump;
      const hitPressed = keys['z'] || touchMap.hit;
      const superPressed = keys['x'] || touchMap.super;

      // أفقي
      if(left && !right){ player.vx = -player.speed; player.facing = -1; }
      else if(right && !left){ player.vx = player.speed; player.facing = 1; }
      else player.vx *= 0.8;

      // قفز — يسمح للقفز إذا على الأرض
      if(jumpPressed && player.onGround){
        player.vy = -14.5;
        player.onGround = false;
      }

      // تطبيق الجاذبية
      player.vy += gravity * 0.9;
      player.x += player.vx;
      player.y += player.vy;

      // حدود المشهد
      if(player.x < 10) player.x = 10;
      if(player.x + player.w > W-10) player.x = W - 10 - player.w;
      if(player.y > H + 200){
        // سقط — يخسر حياة ونقفز إلى سطح
        state.playerHP -= 20;
        player.x = 100; player.y = H - 160; player.vy = 0;
        if(state.playerHP <= 0) gameOver();
      }

      // تداخل مع المنصات (أرض)
      player.onGround = false;
      for(const p of platforms){
        if(player.x + player.w > p.x && player.x < p.x + p.w){
          // تحقق من الاصطدام من أعلى
          if(player.y + player.h > p.y && player.y + player.h - player.vy <= p.y){
            player.y = p.y - player.h;
            player.vy = 0;
            player.onGround = true;
          }
        }
      }

      // هجوم عادي
      if(player.attackCooldown>0) player.attackCooldown--;
      if((hitPressed || keys['k']) && player.attackCooldown===0){
        player.attackCooldown = 18; // تباطؤ الضرب
        player.attacking = true;
        // منطقة الضربة أمام اللاعب
        const range = { x: player.x + (player.facing>0? player.w : -40), y: player.y+6, w:40, h:player.h-10 };
        // تصادم مع الأعداء
        enemies.forEach(e=>{
          if(e.alive && rectsIntersect(range,e)){
            e.hp -= 14;
            flash(e.x + e.w/2, e.y + e.h/2, 18, 'white');
            state.score += 7;
            // تكتسب شريط القوة عند ضرب الأعداء
            player.superPower = Math.min(100, player.superPower + 6);
          }
        });
      } else {
        player.attacking = false;
      }

      // قوة خارقة
      if(superPressed && player.superReady()){
        // انفجار منطقة كبير يضرب كل الأعداء في مدى
        player.superPower = 0;
        flash(player.x + player.w/2, player.y + player.h/2, 80, 'cyan');
        enemies.forEach(e=>{
          const dx = (e.x + e.w/2) - (player.x + player.w/2);
          const dy = (e.y + e.h/2) - (player.y + player.h/2);
          const dist = Math.sqrt(dx*dx + dy*dy);
          if(dist < 260){
            e.hp -= 60 + Math.random()*40;
            state.score += 30;
          }
        });
      }

      // تحديث الأعداء
      enemies.forEach(e=>{
        if(!e.alive) return;
        // سلوك بسيط: يقترب من اللاعب أفقياً
        const dx = (player.x) - e.x;
        e.vx = Math.sign(dx) * e.speed;
        // عشوائية لتغيير الاتجاه مؤقتاً
        e.flipTimer -= 1;
        if(e.flipTimer <= 0){
          e.flipTimer = 60 + Math.random()*120;
          if(Math.random() < 0.12) e.vx *= -1;
        }
        // حركة رأسية وجاذبية
        e.vy += gravity*0.9;
        e.x += e.vx;
        e.y += e.vy;

        // اصطدام بالأرض
        e.onGround = false;
        for(const p of platforms){
          if(e.x + e.w > p.x && e.x < p.x + p.w){
            if(e.y + e.h > p.y && e.y + e.h - e.vy <= p.y){
              e.y = p.y - e.h;
              e.vy = 0;
              e.onGround = true;
            }
          }
        }

        // تصادم مع اللاعب — يهاجم إذا قريب
        if(rectsIntersect(e, player) && e.alive){
          // الضرر مع تبريد
          if(!e._lastHit || Date.now() - e._lastHit > 700){
            state.playerHP -= 8;
            e._lastHit = Date.now();
            flash(player.x + player.w/2, player.y + player.h/2, 20, 'red');
            if(state.playerHP <= 0) gameOver();
          }
        }

        // وفاة العدو
        if(e.hp <= 0){
          e.alive = false;
          flash(e.x + e.w/2, e.y + e.h/2, 40, '#ffd166');
          state.score += 20;
          // بعد موت العدو نولد عدو جديد بعد فترة
          setTimeout(()=> spawnEnemy(840, H - 140, Math.random()>0.6?2:1), 1100 + Math.random()*1200);
        }
      });

      // تحديث المؤثرات
      for(let i=effects.length-1;i>=0;i--){
        effects[i].t++;
        if(effects[i].t > effects[i].d) effects.splice(i,1);
      }

      // زيادة تدريجية للشريط عند النقاط أو الفعاليات — هنا نعطي بمرور الوقت قليل
      player.superPower = Math.min(100, player.superPower + 0.02);

      // تحديث الواجهة
      scoreEl.textContent = Math.floor(state.score);
      hpEl.textContent = Math.max(0, Math.floor(state.playerHP));
      superBarEl.style.width = (player.superPower)+'%';
    }

    // رسم
    function draw(){
      // خلفية
      ctx.clearRect(0,0,W,H);
      // سماء/تلبيس
      // أرض متدرجة
      ctx.fillStyle = '#071126';
      ctx.fillRect(0,0,W,H);

      // منصات
      for(const p of platforms){
        // ظل بسيط
        ctx.fillStyle = '#071e28';
        ctx.fillRect(p.x, p.y, p.w, p.h);
        ctx.fillStyle = '#163447';
        ctx.fillRect(p.x+2, p.y+2, p.w-4, p.h-4);
      }

      // اللاعب
      // جسم
      ctx.save();
      ctx.translate(player.x + player.w/2, player.y + player.h/2);
      // تأثير صغير عندما يهاجم
      let shake = player.attacking ? (Math.random()*2-1)*2 : 0;
      ctx.translate(shake,0);
      ctx.fillStyle = '#f4d35e';
      ctx.fillRect(-player.w/2, -player.h/2, player.w, player.h);
      // عينان بسيطة تحدد الوجه
      ctx.fillStyle = '#2b2f3a';
      ctx.fillRect(4, -12, 8, 8);
      ctx.fillRect(-12, -12, 8, 8);
      ctx.restore();

      // مظاهر القوة الخارقة (هالة) عندما تكون جاهزة
      if(player.superReady()){
        ctx.beginPath();
        ctx.arc(player.x + player.w/2, player.y + player.h/2, 42 + Math.sin(Date.now()/120)*4, 0, Math.PI*2);
        ctx.strokeStyle = 'rgba(6,214,160,0.18)';
        ctx.lineWidth = 10;
        ctx.stroke();
      }

      // أعداء
      enemies.forEach(e=>{
        if(!e.alive) return;
        // جسم
        ctx.fillStyle = '#ef476f';
        ctx.fillRect(e.x, e.y, e.w, e.h);
        // HP bar فوق العدو
        ctx.fillStyle = '#222';
        ctx.fillRect(e.x, e.y-8, e.w, 6);
        const w = Math.max(0, e.hp) / (e.type===1?30:60) * e.w;
        ctx.fillStyle = '#ffd166';
        ctx.fillRect(e.x, e.y-8, w, 6);
      });

      // المؤثرات
      effects.forEach(e=>{
        const alpha = 1 - e.t / e.d;
        ctx.beginPath();
        ctx.arc(e.x, e.y, e.r * (e.t/e.d + 0.2), 0, Math.PI*2);
        ctx.fillStyle = (e.color || 'white');
        ctx.globalAlpha = alpha*0.9;
        ctx.fill();
        ctx.globalAlpha = 1;
      });

      // HUD نصي صغير
      ctx.fillStyle = 'rgba(255,255,255,0.04)';
      ctx.fillRect(8,8,220,46);
      ctx.fillStyle = '#cfe8dd';
      ctx.font = '14px system-ui';
      ctx.fillText('قوة خارقة: ' + Math.floor(player.superPower) + '%', 18, 28);
      ctx.fillText('حياة: ' + Math.floor(state.playerHP), 18, 46);
    }

    function loop(){
      update();
      draw();
      requestAnimationFrame(loop);
    }

    function gameOver(){
      state.running = false;
      // عرض رسالة داخل الكانفس
      ctx.fillStyle = 'rgba(0,0,0,0.6)';
      ctx.fillRect(0,0,W,H);
      ctx.fillStyle = '#fff';
      ctx.font = '36px system-ui';
      ctx.textAlign = 'center';
      ctx.fillText('خسرت! انتهت الحياة', W/2, H/2 - 10);
      ctx.font = '20px system-ui';
      ctx.fillText('اضغط "إعادة تشغيل" لتجربة أخرى', W/2, H/2 + 24);
    }

    btnRestart.addEventListener('click', ()=>{
      // إعادة تهيئة الحالة
      state.running = true;
      state.score = 0;
      state.playerHP = 100;
      player.x = 100; player.y = H - 140; player.vx = 0; player.vy = 0; player.superPower = 0;
      enemies.length = 0;
      for(let i=0;i<4;i++) spawnEnemy(420 + i*120, H - 140, Math.random()>0.7?2:1);
      loop();
    });

    // نبدأ الحلقة
    loop();

  })();
  </script>
</body>
</html>-
قاتل شوارع 
