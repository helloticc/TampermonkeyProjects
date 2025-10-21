(() => {
  /* FASTSIM-PRO v4 — Hybrid Speed/Safe/AI Core
     - Run only on pages you own / local testing.
     - Adds: Dynamic Core Balancer, Infinity Core v2 (microtask queue), Anti-Freeze & Memory Clean.
  */

  const NAME = "FASTSIM-PRO v4 (Hybrid)";
  const VERSION = "v4.0";

  // --- config (tweakable) ---
  const CONF = {
    concurrencyMin: 1,
    concurrencyMax: Math.max(2, navigator.hardwareConcurrency ? Math.min(8, navigator.hardwareConcurrency) : 4),
    concurrencyInit: 3,
    batchDelayMin: 20,
    batchDelayMax: 600,
    batchDelayInit: 200,
    preferInstantReadyState: 2,
    skipStepBase: 1e12,       // base huge step
    skipStepMax: 1e18,
    skipTickMs: 20,           // not used in microtask core but as fallback
    adaptWindowMs: 2000,      // window to measure workms
    adaptTargetWorkMs: 12,    // target ms per tick to keep snappy
    cleanupAfterFinish: true, // try to release memory (safe best-effort)
    cleanupOnlyBlobSrc: true, // only revoke blob: URLs by default (safer)
    toastOnDone: false,       // minimal toast when done (off by default)
    silent: true              // reduce console spam
  };

  // --- internal state ---
  const state = {
    running: true,
    concurrency: CONF.concurrencyInit,
    batchDelay: CONF.batchDelayInit,
    skipStep: CONF.skipStepBase,
    queue: [],            // video elements queued for processing
    processingCount: 0,
    doneCount: 0,
    seenCount: 0,
    readySuccess: 0,      // count of times instant was usable
    readyFailure: 0,
    lastWorkSamples: [],  // ms samples
    observer: null,
    healTimer: null,
    taskScheduled: false,
    apiKey: Symbol("FASTSIM_PRO_V4")
  };

  // --- logger (silent by default) ---
  const lg = (...args) => { if (!CONF.silent) console.log(`[${NAME}]`, ...args); };
  const err = (...args) => { if (!CONF.silent) console.error(`[${NAME}]`, ...args); };

  // --- UTIL: find videos (shadow/root/iframes same-origin) ---
  function findVideos(root = document, out = []) {
    try {
      if (!root) return out;
      if (root.querySelectorAll) {
        const vids = Array.from(root.querySelectorAll("video"));
        for (const v of vids) if (!out.includes(v)) out.push(v);
        // shadow roots
        const all = Array.from(root.querySelectorAll("*"));
        for (const el of all) {
          try { if (el.shadowRoot) findVideos(el.shadowRoot, out); } catch(e){}
        }
        // same-origin iframes
        const frames = Array.from(root.querySelectorAll("iframe"));
        for (const fr of frames) {
          try { if (fr.contentDocument) findVideos(fr.contentDocument, out); } catch(e){}
        }
      }
    } catch(e){}
    return out;
  }

  // --- CLEANUP: try release memory after finish ---
  function cleanupVideoResources(v) {
    try {
      // revoke blob: URLs
      const src = (v && (v.currentSrc || v.src || ""));
      if (typeof src === "string" && src.startsWith("blob:")) {
        try { URL.revokeObjectURL(src); } catch(e){}
        try { v.removeAttribute("src"); } catch(e){}
      }
      // release srcObject if present (MediaStream)
      try {
        if (v.srcObject) {
          try {
            if (v.srcObject.getTracks) {
              v.srcObject.getTracks().forEach(t => { try { t.stop(); } catch(e){} });
            }
          } catch(e){}
          try { v.srcObject = null; } catch(e){}
        }
      } catch(e){}
      // if configured to be aggressive, and it's safe (blob or same-origin), clear src
      if (CONF.cleanupAfterFinish && (!CONF.cleanupOnlyBlobSrc || (typeof src === "string" && src.startsWith("blob:")))) {
        try { v.removeAttribute("src"); } catch(e){}
        try { v.load && v.load(); } catch(e){}
      }
    } catch(e){}
  }

  // minimal toast
  function showToast(msg) {
    if (!CONF.toastOnDone) return;
    try {
      const id = "__fsim_toast_v4";
      let el = document.getElementById(id);
      if (!el) {
        el = document.createElement("div");
        el.id = id;
        Object.assign(el.style, {
          position: "fixed", right: "18px", bottom: "18px",
          background: "rgba(0,0,0,0.6)", color: "#eaffea",
          padding: "8px 12px", borderRadius: "8px",
          zIndex: 2147483647, fontFamily: "system-ui,Segoe UI,Roboto,Arial",
          fontSize: "12px", pointerEvents: "none", opacity: "0", transition: "opacity 260ms"
        });
        document.documentElement.appendChild(el);
      }
      el.textContent = msg;
      requestAnimationFrame(() => el.style.opacity = "1");
      setTimeout(() => { el.style.opacity = "0"; setTimeout(()=>{ try{ el.remove() }catch{} }, 320); }, 1200);
    } catch(e){}
  }

  // --- INFINITY CORE v2: microtask-based processing
  // We maintain a queue of videos to process and drain it in microtask batches,
  // but limit per-batch work (by concurrency) so we don't freeze the main thread.
  function scheduleTaskDrain() {
    if (state.taskScheduled) return;
    state.taskScheduled = true;
    // prefer queueMicrotask if available for microtask scheduling
    const queueFn = (typeof queueMicrotask === "function") ? queueMicrotask : (fn => Promise.resolve().then(fn));
    queueFn(drainTaskBatch);
  }

  function drainTaskBatch() {
    const t0 = performance.now();
    try {
      let processed = 0;
      // process up to current concurrency
      while (state.queue.length > 0 && state.processingCount < state.concurrency && processed < state.concurrency) {
        const v = state.queue.shift();
        if (!v) break;
        // double-check not already done
        if (v.__fsim_done) continue;
        state.processingCount++;
        processed++;
        // process using hybrid decision
        processOneHybrid(v).finally(() => {
          state.processingCount = Math.max(0, state.processingCount - 1);
          // schedule another drain ASAP if queue remains
          if (state.queue.length > 0) scheduleTaskDrain();
        });
      }
    } catch(e) {
      err("drain error", e);
    } finally {
      state.taskScheduled = false;
      const t1 = performance.now();
      const work = Math.max(0, t1 - t0);
      // store sample
      state.lastWorkSamples.push({ t: Date.now(), ms: work });
      // keep only recent window
      const cutoff = Date.now() - CONF.adaptWindowMs;
      state.lastWorkSamples = state.lastWorkSamples.filter(s => s.t >= cutoff);
      adaptCoreParameters();
    }
  }

  // --- HYBRID decision + processing for a single video ---
  async function processOneHybrid(video) {
    const start = performance.now();
    let usedInstant = false;
    try {
      // prefer instant if ready and predictor suggests
      const ready = (typeof video.readyState === "number") ? video.readyState >= CONF.preferInstantReadyState : true;
      // simple predictor: if past majority were ready, prefer instant
      const readyRate = (state.readySuccess + 1) / (state.readySuccess + state.readyFailure + 2);
      const preferInstant = ready && (readyRate > 0.4); // threshold heuristic

      if (ready && preferInstant) {
        // instant jump (Infinity real)
        try { video.muted = true; } catch(e){}
        try { video.pause(); } catch(e){}
        try {
          if (isFinite(video.duration) && video.duration > 0) video.currentTime = Math.max(0, video.duration - 0.001);
          else video.currentTime = Number.MAX_VALUE;
          // dispatch events
          ["timeupdate","seeked","pause","ended"].forEach(ev => {
            try { video.dispatchEvent(new Event(ev, { bubbles: true })); } catch(e){}
          });
          usedInstant = true;
          state.readySuccess++;
        } catch(e) {
          // fallback to skip if instant failed
          state.readyFailure++;
        }
      } else {
        // skip fallback using micro-batched stepping
        // compute adaptive step
        const step = Math.min(CONF.skipStepBase || CONF.skipStepBase, state.skipStep || CONF.skipStepBase, CONF.skipStepMax);
        // do iterative steps but yield periodically
        let done = false;
        while (!done && state.running) {
          try {
            if (!isFinite(video.duration) || video.duration <= 0) {
              video.currentTime = (video.currentTime || 0) + step;
            } else {
              const remain = video.duration - (video.currentTime || 0);
              if (remain <= 0.5) {
                video.currentTime = Math.max(0, video.duration - 0.001);
                try { video.dispatchEvent(new Event("ended", { bubbles: true })); } catch(e){}
                done = true;
                break;
              } else {
                video.currentTime = Math.min(video.duration, (video.currentTime || 0) + Math.min(step, remain));
              }
            }
          } catch(e){ break; }
          // yield to microtask queue once per loop iteration (prevents 100% block)
          await new Promise(res => setTimeout(res, 0));
        }
      }
      // mark done and cleanup
      video.__fsim_done = true;
      state.doneCount++;
      cleanupVideoResources(video);
      showToast(`Done: ${state.doneCount}`);
      return true;
    } catch(e) {
      err("processOneHybrid error", e);
      return false;
    } finally {
      const took = Math.max(0, Math.round(performance.now() - start));
      // update lastWorkSamples quickly
      state.lastWorkSamples.push({ t: Date.now(), ms: took });
      // keep window
      const cutoff = Date.now() - CONF.adaptWindowMs;
      state.lastWorkSamples = state.lastWorkSamples.filter(s => s.t >= cutoff);
    }
  }

  // --- queuing loop: collects videos and schedules microtask drain ---
  function enqueueNewVideos() {
    try {
      const vids = findVideos(document);
      for (const v of vids) {
        if (!v) continue;
        if (v.__fsim_done) continue;
        // avoid duplicates in queue
        if (state.queue.indexOf(v) === -1) state.queue.push(v);
      }
      // ensure totalSeen
      state.seenCount = Math.max(state.seenCount, state.queue.length + state.processingCount + state.doneCount);
      if (state.queue.length > 0) scheduleTaskDrain();
    } catch(e) {}
  }

  // --- Dynamic Core Balancer: adjust concurrency & skipStep & batchDelay based on recent workms ---
  function adaptCoreParameters() {
    try {
      // compute average workms
      const arr = state.lastWorkSamples.map(s => s.ms).filter(Boolean);
      const avg = arr.length ? (arr.reduce((a,b)=>a+b,0)/arr.length) : 0;
      // adapt concurrency: if avg work low, increase concurrency, else reduce
      if (avg < CONF.adaptTargetWorkMs) {
        state.concurrency = Math.min(CONF.concurrencyMax, state.concurrency + 1);
      } else if (avg > CONF.adaptTargetWorkMs * 1.8) {
        state.concurrency = Math.max(CONF.concurrencyMin, state.concurrency - 1);
      }
      // adapt skipStep: if we see many readySuccess -> reduce skipStep (instant favored)
      const readyRate = (state.readySuccess + 1) / (state.readySuccess + state.readyFailure + 2);
      // if readyRate high, we can be more aggressive with instant, but reduce skip steps to save work
      state.skipStep = readyRate > 0.6 ? Math.max(CONF.skipStepBase/10, CONF.skipStepBase/100) : CONF.skipStepBase;
      // adapt batchDelay heuristics (not widely used here, but keep for safe)
      if (avg > CONF.adaptTargetWorkMs * 2) state.batchDelay = Math.min(CONF.batchDelayMax, (state.batchDelay || CONF.batchDelayInit) * 1.5);
      else state.batchDelay = Math.max(CONF.batchDelayMin, (state.batchDelay || CONF.batchDelayInit) * 0.9);
      // clamp concurrency
      state.concurrency = Math.max(CONF.concurrencyMin, Math.min(CONF.concurrencyMax, Math.round(state.concurrency)));
    } catch(e){}
  }

  // --- Observer & Self-heal ---
  function installObserver() {
    try {
      if (state.observer) try { state.observer.disconnect(); } catch(e){}
      state.observer = new MutationObserver(muts => {
        let added = false;
        for (const m of muts) { if (m.addedNodes && m.addedNodes.length) { added = true; break; } }
        if (added) {
          enqueueNewVideos();
        }
      });
      state.observer.observe(document, { childList: true, subtree: true });
    } catch(e) { err("observer install err", e); }
  }

  function startSelfHeal() {
    if (state.healTimer) clearInterval(state.healTimer);
    state.healTimer = setInterval(() => {
      try {
        if (!state.running) return;
        if (!state.observer) installObserver();
      } catch(e){}
    }, 3000);
  }

  // --- public API (non-enumerable) ---
  function stopAll() {
    state.running = false;
    try { if (state.observer) { state.observer.disconnect(); state.observer = null; } } catch(e){}
    try { if (state.healTimer) { clearInterval(state.healTimer); state.healTimer = null; } } catch(e){}
    // clear queue references, but avoid touching page video elements too aggressively
    state.queue.length = 0;
    lg("stopped");
  }
  function status() {
    return {
      running: state.running,
      concurrency: state.concurrency,
      queueLength: state.queue.length,
      processing: state.processingCount,
      done: state.doneCount,
      seen: state.seenCount,
      avgWorkMs: state.lastWorkSamples.length ? (state.lastWorkSamples.reduce((a,b)=>a+b.ms,0)/state.lastWorkSamples.length).toFixed(2) : 0
    };
  }

  try {
    Object.defineProperty(window, "__FASTSIM_PRO_V4__", {
      configurable: true,
      enumerable: false,
      value: {
        stop: stopAll,
        status,
        config: CONF,
        rawState: state
      }
    });
  } catch(e) {
    window.__FASTSIM_PRO_V4__ = { stop: stopAll, status, config: CONF, rawState: state };
  }

  // --- bootstrap ---
  installObserver();
  startSelfHeal();
  // initial enqueue & start drain loop
  enqueueNewVideos();
  // schedule periodic enqueue to catch soft-loaded videos
  const periodicEnq = setInterval(() => { if (!state.running) { clearInterval(periodicEnq); return; } enqueueNewVideos(); }, Math.max(200, CONF.batchDelayInit));
  // also schedule occasional microtask drains to ensure timely processing
  const periodicDrain = setInterval(() => { if (!state.running) { clearInterval(periodicDrain); return; } if (state.queue.length) scheduleTaskDrain(); }, 120);

  lg(`${NAME} ${VERSION} booted — concurrency:${state.concurrency}`);
})();
