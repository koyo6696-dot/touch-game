<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>順番タッチゲーム</title>
  <style>
    * { margin:0; padding:0; box-sizing:border-box; }
    body{
      font-family: Arial, sans-serif;
      background: linear-gradient(135deg,#f093fb 0%,#f5576c 100%);
      min-height:100vh;
      display:flex; justify-content:center; align-items:center;
      user-select:none; -webkit-user-select:none;
      touch-action: manipulation;
      overflow:hidden;
    }
    .game-container{
      background:white; border-radius:20px; padding:30px;
      box-shadow:0 20px 60px rgba(0,0,0,0.3);
      text-align:center; max-width:600px; width:90%;
      position:relative; z-index:1;
    }
    h1{ color:#f5576c; margin-bottom:20px; font-size:28px; }
    .score{ font-size:24px; color:#333; margin-bottom:20px; font-weight:bold; }
    .score-number{
      font-size:80px; color:#f5576c; font-weight:bold; display:inline-block;
      text-shadow:3px 3px 6px rgba(0,0,0,0.2);
    }
    .score-grow{ animation: scoreGrow .5s ease-out; }
    @keyframes scoreGrow{ 0%{transform:scale(1)} 50%{transform:scale(1.5); color:#ffd700} 100%{transform:scale(1)} }
    .game-area{
      background:#f0f0f0; border-radius:15px; padding:20px;
      margin:20px 0; min-height:400px;
      display:flex; flex-direction:column; justify-content:center; align-items:center;
      position:relative;
    }
    .numbers-container{ position:relative; width:100%; max-width:500px; height:350px; margin:0 auto; }
    .number-box{
      position:absolute;
      background:white; border:4px solid #ddd; border-radius:15px;
      width:70px; height:70px;
      display:flex; justify-content:center; align-items:center;
      font-size:40px; font-weight:bold; color:#333;
      cursor:pointer;
      transition:all .2s;
      box-shadow:0 4px 8px rgba(0,0,0,0.1);
      -webkit-tap-highlight-color: transparent;
    }
    .number-box:active{ transform:scale(.9); }
    .number-box.correct{
      background:linear-gradient(135deg,#27ae60 0%,#2ecc71 100%);
      border-color:#27ae60; color:white; animation: correctPulse .25s;
    }
    .number-box.wrong{
      background:linear-gradient(135deg,#e74c3c 0%,#c0392b 100%);
      border-color:#e74c3c; color:white; animation: shake .4s;
    }
    @keyframes correctPulse{ 0%{transform:scale(1)} 50%{transform:scale(1.15)} 100%{transform:scale(1)} }
    @keyframes shake{ 0%,100%{transform:translateX(0)} 25%{transform:translateX(-10px)} 75%{transform:translateX(10px)} }

    .instruction{ font-size:32px; font-weight:bold; color:#f5576c; margin-bottom:20px; min-height:40px; }
    .feedback{ font-size:48px; font-weight:bold; margin-top:20px; min-height:60px; }
    .success{ color:#27ae60; animation: successBounce .6s ease-out; }
    .error{ color:#e74c3c; }
    @keyframes successBounce{ 0%{transform:scale(0)} 50%{transform:scale(1.3)} 100%{transform:scale(1)} }

    button{
      background: linear-gradient(135deg,#f093fb 0%,#f5576c 100%);
      color:white; border:none; padding:15px 40px; font-size:24px;
      border-radius:50px; cursor:pointer;
      transition: transform .2s, box-shadow .2s;
      font-weight:bold; margin-top:20px;
    }
    button:hover{ transform:translateY(-2px); box-shadow:0 5px 15px rgba(0,0,0,0.2); }
    button:active{ transform:translateY(0); }
    .instructions{ color:#666; font-size:16px; margin-top:15px; line-height:1.6; }

    .confetti{ position:fixed; width:10px; height:10px; background:#f0f; animation: confetti-fall 3s linear forwards; }
    @keyframes confetti-fall{ to{ transform:translateY(100vh) rotate(360deg); opacity:0; } }
    .star{ position:fixed; font-size:30px; animation: star-burst 1s ease-out forwards; }
    @keyframes star-burst{
      0%{ transform:translate(0,0) scale(0); opacity:1; }
      100%{ transform:translate(var(--tx),var(--ty)) scale(1); opacity:0; }
    }
    .flash{ position:fixed; top:0; left:0; width:100vw; height:100vh; background:rgba(255,255,255,.8);
      pointer-events:none; animation: flash .5s ease-out; z-index:999;
    }
    @keyframes flash{ 0%{opacity:0} 50%{opacity:1} 100%{opacity:0} }
  </style>
</head>
<body>
  <div class="game-container">
    <h1>🔢 順番タッチゲーム</h1>
    <div class="score">最高記録: <span class="score-number" id="score">2</span></div>

    <div class="game-area" id="gameArea">
      <div id="startScreen">
        <button id="startBtn">スタート</button>
        <div class="instructions">
          1から順番にタッチしよう！<br>正しい順番でタッチできるかな？
        </div>
      </div>
    </div>

    <div class="feedback" id="feedback"></div>
  </div>

  <script>
    let maxScore = 2;
    let currentLevel = 2;
    let playerSequence = [];
    let canClick = false;

    // ✅ AudioContextは使い回し（iOS対策）
    let audioCtx = null;
    function ensureAudio() {
      if (!audioCtx) audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      if (audioCtx.state === "suspended") audioCtx.resume();
    }
    function beep(freq, duration, gain=0.25, type="sine") {
      if (!audioCtx) return;
      const o = audioCtx.createOscillator();
      const g = audioCtx.createGain();
      o.type = type;
      o.frequency.value = freq;
      o.connect(g); g.connect(audioCtx.destination);
      g.gain.setValueAtTime(gain, audioCtx.currentTime);
      g.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
      o.start();
      o.stop(audioCtx.currentTime + duration);
    }
    function playClickSound(){ ensureAudio(); beep(600, 0.08, 0.20); }
    function playSuccessSound(){
      ensureAudio();
      beep(800, 0.25, 0.25);
      setTimeout(()=>beep(1000, 0.25, 0.25), 90);
    }
    function playWrongSound(){ ensureAudio(); beep(180, 0.25, 0.25, "square"); }

    function createConfetti(){
      const colors=['#ff0000','#00ff00','#0000ff','#ffff00','#ff00ff','#00ffff','#ffa500','#ff1493'];
      for (let i=0;i<80;i++){
        setTimeout(()=>{
          const c=document.createElement('div');
          c.className='confetti';
          c.style.left=Math.random()*100+'%';
          c.style.top='-10px';
          c.style.background=colors[(Math.random()*colors.length)|0];
          c.style.width=(Math.random()*10+5)+'px';
          c.style.height=(Math.random()*10+5)+'px';
          c.style.animationDuration=(Math.random()*2+2)+'s';
          c.style.animationDelay=(Math.random()*0.5)+'s';
          document.body.appendChild(c);
          setTimeout(()=>c.remove(), 3500);
        }, i*15);
      }
    }
    function createStars(){
      const n=30, cx=innerWidth/2, cy=innerHeight/2;
      for(let i=0;i<n;i++){
        const s=document.createElement('div');
        s.className='star';
        s.textContent='⭐';
        s.style.left=cx+'px';
        s.style.top=cy+'px';
        const a=(Math.PI*2*i)/n;
        const d=200+Math.random()*200;
        s.style.setProperty('--tx', (Math.cos(a)*d)+'px');
        s.style.setProperty('--ty', (Math.sin(a)*d)+'px');
        document.body.appendChild(s);
        setTimeout(()=>s.remove(), 1000);
      }
    }
    function createFlash(){
      const f=document.createElement('div');
      f.className='flash';
      document.body.appendChild(f);
      setTimeout(()=>f.remove(), 500);
    }

    // ✅ 重なり判定：矩形（AABB）でチェック
    function overlaps(r1, r2, pad){
      return !(
        r1.x + r1.w + pad <= r2.x ||
        r2.x + r2.w + pad <= r1.x ||
        r1.y + r1.h + pad <= r2.y ||
        r2.y + r2.h + pad <= r1.y
      );
    }

    function getRandomPositions(count){
      const positions=[];
      const box=70, pad=12, margin=10;
      const container=document.querySelector('.numbers-container');
      const W=container ? container.offsetWidth : 500;
      const H=350;
      const maxX=W - box - margin;
      const maxY=H - box - margin;

      for(let i=0;i<count;i++){
        let ok=false, x=margin, y=margin;
        for(let attempt=0; attempt<200 && !ok; attempt++){
          x = Math.random()*maxX + margin;
          y = Math.random()*maxY + margin;
          const rect={x,y,w:box,h:box};
          ok = positions.every(p => !overlaps(rect, {x:p.x,y:p.y,w:box,h:box}, pad));
        }
        if(!ok){
          // フォールバック：ゆるいグリッド
          const cols=Math.ceil(Math.sqrt(count));
          const row=(i/cols)|0, col=i%cols;
          x = margin + col*(W/cols) ;
          y = margin + row*(H/Math.ceil(count/cols));
        }
        positions.push({x,y});
      }
      return positions;
    }

    function updateScore(){
      const el=document.getElementById('score');
      el.textContent=maxScore;
      el.classList.remove('score-grow');
      void el.offsetWidth;
      el.classList.add('score-grow');
    }

    function startGame(){
      ensureAudio(); // ✅ 初回に解除
      currentLevel = 2;
      playerSequence = [];
      canClick = true;
      nextRound();
    }

    function nextRound(){
      const gameArea=document.getElementById('gameArea');
      const feedback=document.getElementById('feedback');
      feedback.textContent='';
      feedback.className='feedback';

      playerSequence=[];
      canClick=true;

      gameArea.innerHTML =
        '<div class="instruction">1から順番にタッチしてね！</div>' +
        '<div class="numbers-container" id="numbersContainer"></div>';

      setTimeout(()=>{
        const positions=getRandomPositions(currentLevel);
        const container=document.getElementById('numbersContainer');

        for(let i=1;i<=currentLevel;i++){
          const pos=positions[i-1];
          const box=document.createElement('div');
          box.className='number-box';
          box.id='num'+i;
          box.textContent=i;
          box.style.left=pos.x+'px';
          box.style.top=pos.y+'px';

          // ✅ pointerdownで反応安定（多指でも暴発しにくい）
          box.addEventListener('pointerdown', (e)=>{
            e.preventDefault();
            handleNumberClick(i);
          }, {passive:false});

          container.appendChild(box);
        }
      }, 10);
    }

    function handleNumberClick(num){
      if(!canClick) return;
      playClickSound();

      const expected = playerSequence.length + 1;
      const box = document.getElementById('num'+num);

      if(num === expected){
        playerSequence.push(num);
        box.classList.add('correct');

        if(playerSequence.length === currentLevel){
          canClick=false;
          setTimeout(handleSuccess, 450);
        }
      }else{
        box.classList.add('wrong');
        canClick=false;
        playWrongSound();
        handleFailure();
      }
    }

    function handleSuccess(){
      const feedback=document.getElementById('feedback');

      playSuccessSound();
      createConfetti();
      createStars();
      createFlash();

      const messages=['すごい！✨','できた！🎉','かんぺき！🌟','やったね！🎊','天才！👏'];
      feedback.textContent=messages[(Math.random()*messages.length)|0];
      feedback.className='feedback success';

      // ✅ 最高記録は「達成した数」で更新（+1バグ防止）
      maxScore = Math.max(maxScore, currentLevel);
      updateScore();

      currentLevel++;

      setTimeout(nextRound, 2000);
    }

    function handleFailure(){
      const feedback=document.getElementById('feedback');
      feedback.textContent='もう一度がんばろう！';
      feedback.className='feedback error';

      setTimeout(()=>{
        currentLevel=2;
        nextRound();
      }, 1400);
    }

    document.getElementById('startBtn').addEventListener('click', startGame);
  </script>
</body>
</html>