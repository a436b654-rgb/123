# 123
123
const $ = (s) => document.querySelector(s);
const today = new Date().toISOString().slice(0, 10);
$('#date').value = today;
$('#reportDate').textContent = today;
$('#reportDate2').textContent = today;

const modal = $('#safetyModal');
const toast = $('#toast');
function showToast(message) {
  toast.textContent = message;
  toast.classList.add('show');
  setTimeout(() => toast.classList.remove('show'), 2400);
}

$('#safetyConfirm').addEventListener('change', (event) => {
  $('#enterBtn').disabled = !event.target.checked;
  $('#enterBtn').textContent = event.target.checked ? '進入 JumpLab' : '請先勾選總確認';
});
$('#enterBtn').addEventListener('click', () => { modal.style.display = 'none'; });
$('#helpBtn').addEventListener('click', () => { modal.style.display = 'grid'; });

const video = $('#video');
const fileInput = $('#videoInput');
const controls = ['clipStart','clipEnd','timeline','analyzeBtn','skeletonBtn','toeBtn','landBtn','back10Btn','backBtn','playBtn','nextBtn','next10Btn','saveBtn'];
let duration = 0;
let frameRate = 240;
let analyzed = false;
let toeFrame = null;
let landFrame = null;
let recordedStream = null;
let recorder = null;
let recordedChunks = [];

function formatTime(seconds) {
  if (!Number.isFinite(seconds)) return '00:00.000';
  const min = Math.floor(seconds / 60);
  const sec = seconds % 60;
  return `${String(min).padStart(2,'0')}:${sec.toFixed(3).padStart(6,'0')}`;
}
function setControls(enabled) {
  controls.forEach(id => { $('#' + id).disabled = !enabled; });
  $('#clipEnd').value = 1000;
  $('#timeline').max = Math.max(0, Math.floor(duration * frameRate));
}
function updateTimeline() {
  const current = video.currentTime || 0;
  $('#timeline').value = Math.round(current * frameRate);
  $('#frameInfo').innerHTML = `${formatTime(current)}&nbsp; Frame ${Math.round(current * frameRate)} / ${Math.max(1, Math.floor(duration * frameRate))}`;
  $('#frameBadge').textContent = `FRAME ${Math.round(current * frameRate)}`;
}
function loadVideo(file) {
  if (!file) return;
  const url = URL.createObjectURL(file);
  video.src = url;
  video.load();
  video.onloadedmetadata = () => {
    duration = video.duration || 0;
    $('#videoStage').classList.add('has-video');
    $('#videoStatus').firstChild.textContent = `影片  ${file.name} `;
    $('#reportVideo').textContent = file.name;
    setControls(true);
    toeFrame = null;
    landFrame = null;
    document.querySelectorAll('.marker-btn').forEach((button) => button.classList.remove('marked'));
    $('#measurementNote').textContent = '逐格找到「離地」與「首次落地」後，按「計算跳躍高度」。';
    $('#startTime').textContent = formatTime(0);
    $('#endTime').textContent = formatTime(duration);
    $('#keepTime').textContent = formatTime(duration);
    updateTimeline();
    showToast('影片已匯入，可開始逐格檢視');
  };
}
$('#importBtn').addEventListener('click', () => fileInput.click());
$('#importTool').addEventListener('click', () => fileInput.click());
fileInput.addEventListener('change', (event) => loadVideo(event.target.files[0]));
$('#fps').addEventListener('change', (event) => {
  frameRate = Number(event.target.value);
  $('#reportFps').textContent = event.target.value;
  if (duration) { $('#timeline').max = Math.floor(duration * frameRate); updateTimeline(); }
});
video.addEventListener('timeupdate', updateTimeline);
$('#timeline').addEventListener('input', (event) => { video.currentTime = Number(event.target.value) / frameRate; updateTimeline(); });
$('#playBtn').addEventListener('click', () => {
  if (video.paused) { video.play(); $('#playBtn').innerHTML = 'Ⅱ<span>暫停</span>'; }
  else { video.pause(); $('#playBtn').innerHTML = '▶<span>播放</span>'; }
});
function step(frames) { video.pause(); video.currentTime = Math.max(0, Math.min(duration, video.currentTime + frames / frameRate)); updateTimeline(); }
$('#backBtn').addEventListener('click', () => step(-1));
$('#nextBtn').addEventListener('click', () => step(1));
$('#back10Btn').addEventListener('click', () => step(-10));
$('#next10Btn').addEventListener('click', () => step(10));
$('#clipStart').addEventListener('input', (event) => { const t = duration * Number(event.target.value) / 1000; $('#startTime').textContent = formatTime(t); $('#keepTime').textContent = formatTime(Math.max(0, duration - t)); });
$('#clipEnd').addEventListener('input', (event) => { const t = duration * Number(event.target.value) / 1000; $('#endTime').textContent = formatTime(t); $('#keepTime').textContent = formatTime(Math.max(0, t - duration * Number($('#clipStart').value) / 1000)); });

function setMarker(kind) {
  if (!duration) return;
  const frame = Math.round((video.currentTime || 0) * frameRate);
  if (kind === 'toe') {
    toeFrame = frame;
    $('#toeBtn').classList.add('marked');
    showToast(`已標記離地：第 ${frame} 格`);
  } else {
    landFrame = frame;
    $('#landBtn').classList.add('marked');
    showToast(`已標記落地：第 ${frame} 格`);
  }
  $('#measurementNote').textContent = `離地：${toeFrame ?? '—'} 格　·　落地：${landFrame ?? '—'} 格`;
}
$('#toeBtn').addEventListener('click', () => setMarker('toe'));
$('#landBtn').addEventListener('click', () => setMarker('land'));

$('#analyzeBtn').addEventListener('click', () => {
  if (toeFrame === null || landFrame === null || landFrame <= toeFrame) {
    showToast('請先正確標記離地與首次落地影格');
    return;
  }
  analyzed = true;
  const flight = (landFrame - toeFrame) / frameRate;
  // Ballistic estimate: h = g * t² / 8. This is an estimate, not a force-plate measurement.
  const height = (9.81 * flight * flight / 8) * 100;
  ['flightTop','flightResult','reportFlight'].forEach(id => $('#' + id).textContent = flight.toFixed(3));
  ['heightTop','heightResult','reportHeight'].forEach(id => $('#' + id).textContent = height.toFixed(1));
  $('#saveBtn').disabled = false;
  $('#pdfBtn').disabled = false;
  $('#shareBtn').disabled = false;
  $('#shareBtn').textContent = '↗ 分享報告';
  $('#measurementNote').textContent = `以 ${frameRate} FPS、${landFrame - toeFrame} 格滯空時間換算。公式：h = g × t² ÷ 8。`;
  document.querySelector('.confidence').textContent = '估算方式：以逐格標記的滯空時間換算。結果會受 FPS、拍攝角度、動態模糊與標記誤差影響，不能取代測力板或專業量測。';
  showToast(`計算完成：${height.toFixed(1)} cm`);
});
$('#saveBtn').addEventListener('click', () => showToast('已儲存本次 CMJ 檢測（本機示範）'));
$('#copyBtn').addEventListener('click', async () => {
  const text = `JumpLab V3\\nFlight Time: ${$('#flightResult').textContent} s\\nJump Height: ${$('#heightResult').textContent} cm`;
  try { await navigator.clipboard.writeText(text); showToast('結果已複製'); } catch { showToast('瀏覽器未允許複製'); }
});
$('#shareBtn').addEventListener('click', async () => {
  const data = { title: 'JumpLab V3 分析報告', text: `Flight Time ${$('#flightResult').textContent} s · Jump Height ${$('#heightResult').textContent} cm` };
  if (navigator.share) await navigator.share(data); else showToast('目前裝置不支援原生分享');
});
$('#recordBtn').addEventListener('click', async () => {
  if (recorder && recorder.state === 'recording') {
    recorder.stop();
    return;
  }
  if (!navigator.mediaDevices?.getUserMedia || !window.MediaRecorder) {
    showToast('此瀏覽器不支援相機錄影，請改用匯入影片');
    return;
  }
  try {
    recordedStream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' }, audio: false });
    video.srcObject = recordedStream;
    video.muted = true;
    video.play();
    $('#videoStage').classList.add('has-video');
    $('#recordBtn').classList.add('recording');
    $('#recordBtn').querySelector('span').textContent = '停止錄影';
    recordedChunks = [];
    recorder = new MediaRecorder(recordedStream);
    recorder.ondataavailable = (event) => { if (event.data.size) recordedChunks.push(event.data); };
    recorder.onstop = () => {
      recordedStream.getTracks().forEach((track) => track.stop());
      const blob = new Blob(recordedChunks, { type: recorder.mimeType || 'video/webm' });
      const file = new File([blob], `jump-${new Date().toISOString().slice(0,19).replace(/:/g, '-')}.webm`, { type: blob.type });
      video.srcObject = null;
      $('#recordBtn').classList.remove('recording');
      $('#recordBtn').querySelector('span').textContent = '錄影';
      loadVideo(file);
    };
    recorder.start();
    showToast('相機已啟動，完成跳躍後按一次停止錄影');
  } catch {
    showToast('無法取得相機權限，請允許相機存取或改用匯入影片');
  }
});
$('#pdfBtn').addEventListener('click', () => {
  if (!analyzed) return;
  const report = [
    'JumpLab 跳躍測量報告',
    `檢測日期：${$('#date').value || today}`,
    `姓名：${$('#name').value || '—'}`,
    `Flight Time：${$('#flightResult').textContent} s`,
    `Jump Height：${$('#heightResult').textContent} cm`,
    `FPS：${frameRate}`,
    '',
    '本報告為單鏡頭逐格影像估算，請勿視為醫療診斷或測力板結果。'
  ].join('\n');
  const url = URL.createObjectURL(new Blob([report], { type: 'text/plain;charset=utf-8' }));
  const link = document.createElement('a');
  link.href = url;
  link.download = 'jump-lab-report.txt';
  link.click();
  URL.revokeObjectURL(url);
});
$('#testType').addEventListener('change', (event) => showToast(`已切換測驗：${event.target.value}`));
