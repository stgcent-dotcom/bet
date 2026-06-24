<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>LIMITED WATCH — GitHub Edition</title>
<style>
  :root{
    --bg:#0a0d13; --panel:#11161f; --panel2:#0d1219; --border:#1f2a3a;
    --text:#e7ebf2; --muted:#7c8798; --gold:#e8a33d; --gold-dim:#5c4621;
    --green:#43c98a; --green-dim:#173829; --red:#f2545b; --red-dim:#3a1a1d;
  }
  *{box-sizing:border-box;}
  html,body{margin:0;background:var(--bg);color:var(--text);font-family:-apple-system,BlinkMacSystemFont,"Segoe UI",Helvetica,Arial,sans-serif;}
  .mono{font-family:ui-monospace,SFMono-Regular,"SF Mono",Menlo,Consolas,monospace;}
  .condensed{transform:scaleX(0.92);transform-origin:left center;display:inline-block;}
  header{padding:22px 24px 16px;border-bottom:1px solid var(--border);}
  .eyebrow{text-transform:uppercase;font-size:11px;letter-spacing:0.14em;color:var(--gold);font-weight:700;margin-bottom:6px;}
  h1{margin:0;font-size:28px;font-weight:800;letter-spacing:-0.01em;}
  .subhead{color:var(--muted);font-size:13px;margin-top:6px;max-width:680px;line-height:1.5;}
  .controls{display:flex;flex-wrap:wrap;gap:18px;align-items:center;padding:14px 24px;border-bottom:1px solid var(--border);background:var(--panel2);font-size:12.5px;color:var(--muted);}
  .status-dot{width:8px;height:8px;border-radius:50%;display:inline-block;margin-right:6px;background:var(--muted);}
  .status-dot.live{background:var(--green);box-shadow:0 0 6px var(--green);}
  .status-dot.error{background:var(--red);box-shadow:0 0 6px var(--red);}
  button{background:var(--panel);color:var(--text);border:1px solid var(--border);border-radius:6px;padding:7px 14px;font-size:12.5px;cursor:pointer;font-weight:600;}
  button:hover{border-color:var(--gold);}
  main{padding:20px 24px;}
  .section-title{font-size:12px;text-transform:uppercase;letter-spacing:0.1em;color:var(--muted);margin:0 0 12px;font-weight:700;}
  table{width:100%;border-collapse:collapse;font-size:13px;}
  thead th{text-align:left;color:var(--muted);font-weight:600;font-size:11px;text-transform:uppercase;letter-spacing:0.05em;padding:8px 10px;border-bottom:1px solid var(--border);}
  tbody td{padding:10px 10px;border-bottom:1px solid #161c26;}
  tbody tr:hover{background:#141a24;}
  .item-name{font-weight:700;}
  .item-name a{color:inherit;text-decoration:none;}
  .item-name a:hover{text-decoration:underline;color:var(--gold);}
  .score-badge{display:inline-block;min-width:36px;text-align:center;padding:3px 8px;border-radius:5px;font-weight:800;font-size:12.5px;}
  .pct{font-weight:700;}
  .pct.up{color:var(--green);} .pct.down{color:var(--red);} .pct.flat{color:var(--muted);}
  .tag{display:inline-block;font-size:10.5px;padding:2px 6px;border-radius:4px;border:1px solid var(--border);color:var(--muted);margin-right:4px;}
  .empty-state{color:var(--muted);font-size:13px;padding:40px 10px;text-align:center;border:1px dashed var(--border);border-radius:8px;}
  footer{padding:18px 24px 28px;color:var(--muted);font-size:11.5px;line-height:1.6;border-top:1px solid var(--border);}
  .banner{margin:14px 24px 0;padding:10px 14px;border-radius:7px;font-size:12.5px;background:var(--red-dim);border:1px solid var(--red);color:#ffd2d4;display:none;}
  .banner.show{display:block;}
</style>
</head>
<body>
  <header>
    <div class="eyebrow">GitHub Actions feed · updates every ~15 min</div>
    <h1 class="condensed">LIMITED WATCH</h1>
    <p class="subhead">A GitHub Action polls Rolimons server-side, scores every limited, and posts Discord alerts. This page just shows the latest result. <b style="color:var(--text)">Heuristic signal, not financial advice.</b></p>
  </header>

  <div class="banner" id="errorBanner"></div>

  <div class="controls">
    <span><span class="status-dot" id="statusDot"></span><span id="statusText">Loading…</span></span>
    <span>Last run: <b class="mono" id="lastRun" style="color:var(--text)">—</b></span>
    <span>Items tracked: <b class="mono" id="itemCount" style="color:var(--text)">—</b></span>
    <span>Threshold: <b class="mono" id="threshold" style="color:var(--text)">—</b></span>
    <button id="refreshBtn">↻ Refresh</button>
  </div>

  <main>
    <p class="section-title">Top scored limiteds</p>
    <table>
      <thead>
        <tr><th>Item</th><th>Score</th><th>Value</th><th>RAP</th><th>Demand</th><th>Trend</th><th>Δ since last check</th><th>Flags</th></tr>
      </thead>
      <tbody id="oppBody">
        <tr><td colspan="8"><div class="empty-state">Loading latest data…</div></td></tr>
      </tbody>
    </table>
  </main>

  <footer>
    Data: Rolimons public item API, fetched server-side by GitHub Actions on a schedule (no browser CORS issue, since this runs on GitHub's servers). This dashboard is read-only — it doesn't touch your Roblox account or place trades.
  </footer>

<script>
(function(){
  "use strict";
  const $ = id => document.getElementById(id);
  const statusDot = $("statusDot"), statusText = $("statusText"), errorBanner = $("errorBanner");
  const oppBody = $("oppBody"), lastRun = $("lastRun"), itemCount = $("itemCount"), threshold = $("threshold");

  function escapeHtml(s){
    return String(s).replace(/[&<>"']/g, c => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#39;"}[c]));
  }
  function scoreColor(score, thr){
    if (score >= thr) return { bg:"var(--green-dim)", fg:"#9af0c6" };
    if (score <= -3) return { bg:"var(--red-dim)", fg:"#ffb3b7" };
    return { bg:"#1b2230", fg:"var(--muted)" };
  }

  async function load(){
    statusText.textContent = "Loading…";
    statusDot.className = "status-dot";
    try{
      const res = await fetch("data/latest.json?_=" + Date.now()); // cache-bust
      if (!res.ok) throw new Error("HTTP " + res.status);
      const data = await res.json();

      const thr = data.score_threshold ?? 6;
      threshold.textContent = thr;
      itemCount.textContent = (data.item_count ?? 0).toLocaleString();
      lastRun.textContent = data.last_run ? new Date(data.last_run * 1000).toLocaleString() : "never run yet";

      const top = data.top || [];
      if (!top.length){
        oppBody.innerHTML = `<tr><td colspan="8"><div class="empty-state">No data yet — the GitHub Action hasn't completed a run. Check the Actions tab in your repo.</div></td></tr>`;
      }else{
        oppBody.innerHTML = top.map(o => {
          const sc = scoreColor(o.score, thr);
          const momHtml = (o.momentum_pct === null || o.momentum_pct === undefined)
            ? `<span class="pct flat">—</span>`
            : `<span class="pct ${o.momentum_pct >= 0 ? "up":"down"}">${o.momentum_pct >= 0 ? "▲":"▼"} ${Math.abs(o.momentum_pct).toFixed(1)}%</span>`;
          const flags = [
            o.projected ? `<span class="tag" style="color:#ffb3b7;border-color:var(--red)">Projected</span>` : "",
            o.hyped ? `<span class="tag" style="color:#ffd896;border-color:var(--gold)">Hyped</span>` : "",
            o.rare ? `<span class="tag">Rare</span>` : "",
          ].filter(Boolean).join("");
          return `<tr>
            <td class="item-name"><a href="${o.url}" target="_blank" rel="noopener">${escapeHtml(o.name)}</a></td>
            <td><span class="score-badge mono" style="background:${sc.bg};color:${sc.fg}">${o.score}</span></td>
            <td class="mono">${o.value.toLocaleString()}</td>
            <td class="mono">${o.rap.toLocaleString()}</td>
            <td>${o.demand_label}</td>
            <td>${o.trend_label}</td>
            <td>${momHtml}</td>
            <td>${flags || "—"}</td>
          </tr>`;
        }).join("");
      }

      statusDot.className = "status-dot live";
      statusText.textContent = "Live";
      errorBanner.classList.remove("show");
    }catch(err){
      statusDot.className = "status-dot error";
      statusText.textContent = "Error loading data";
      errorBanner.textContent = "Couldn't load data/latest.json — " + err.message + ". Make sure the GitHub Action has run at least once (check the Actions tab) and that GitHub Pages is enabled.";
      errorBanner.classList.add("show");
    }
  }

  $("refreshBtn").addEventListener("click", load);
  load();
  setInterval(load, 2 * 60 * 1000); // auto-refresh every 2 minutes
})();
</script>
</body>
</html>
