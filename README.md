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

$('#analyzeBtn').addEventListener('click', () => {
  analyzed = true;
  const flight = 0.612;
  const height = 45.9;
  ['flightTop','flightResult','reportFlight'].forEach(id => $('#' + id).textContent = flight.toFixed(3));
  ['heightTop','heightResult','reportHeight'].forEach(id => $('#' + id).textContent = height.toFixed(1));
  $('#saveBtn').disabled = false;
  $('#pdfBtn').disabled = false;
  $('#shareBtn').disabled = false;
  $('#shareBtn').textContent = '↗ 分享報告';
  document.querySelector('.confidence').textContent = 'AI 信心： 87%。外骨骼、角度、地板線與關鍵影格離地高度皆為單鏡頭 2D AI 估算，需人工逐格複核。';
  showToast('分析完成：已產生示範結果，可再人工逐格複核');
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
$('#recordBtn').addEventListener('click', () => showToast('錄影功能將使用裝置相機（示範版尚未啟動）'));
$('#testType').addEventListener('change', (event) => showToast(`已切換測驗：${event.target.value}`));
