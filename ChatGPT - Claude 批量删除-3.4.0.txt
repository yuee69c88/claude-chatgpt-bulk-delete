// ==UserScript==
// @name         ChatGPT / Claude 批量删除
// @namespace    local.ai-chat-tools
// @version      3.4.0
// @description  固定悬浮入口，多选、搜索、批量删除，无确认弹窗。
// @match        https://chatgpt.com/*
// @match        https://chat.openai.com/*
// @match        https://claude.ai/*
// @grant        none
// @run-at       document-idle
// @noframes
// @license      MIT
// ==/UserScript==

(() => {
  'use strict';
  if (window.top !== window.self || document.getElementById('chat-cleaner-v2')) return;
  const claude = location.hostname === 'claude.ai';
  const brand = claude ? 'Claude' : 'ChatGPT';
  const CLAUDE_BATCH_SIZE = 500;
  const CLAUDE_CONCURRENCY = 2;
  let items = [], selected = new Set(), busy = false, stopped = false, context = null;
  let nextOffset = 0, hasMore = false;
  const deletedIds = new Set();
  const siteStyle = document.createElement('style');
  siteStyle.textContent = '[data-chat-cleaner-deleted]{display:none!important}';
  document.head.appendChild(siteStyle);
  function chatId(anchor) {
    try {
      const url = new URL(anchor.href, location.origin);
      if (url.origin !== location.origin) return null;
      return url.pathname.match(claude ? /^\/chat\/([\w-]+)\/?$/ : /^\/(?:g\/[^/]+\/)?c\/([\w-]+)\/?$/)?.[1];
    } catch { return null; }
  }
  function syncSidebar() {
    if (!deletedIds.size) return;
    const scope = claude
      ? 'a[href*="/chat/"]'
      : 'nav a[href],aside a[href],[role=navigation] a[href]';
    for (const anchor of document.querySelectorAll(scope)) {
      const id = chatId(anchor);
      if (!deletedIds.has(id)) continue;
      // 保留 React 管理的节点，只隐藏精确对应的条目。
      let row = anchor;
      const parent = anchor.parentElement;
      if (parent && (!claude || parent.matches('li,[role=listitem],[data-sidebar-item]')) && !parent.matches('nav,aside,main,[role=navigation]') &&
          [...parent.querySelectorAll('a[href]')].every(a => chatId(a) === id)) row = parent;
      row.setAttribute('data-chat-cleaner-deleted', '');
    }
  }
  let sidebarTimer;
  function scheduleSidebar() {
    if (sidebarTimer) return;
    sidebarTimer = setTimeout(() => { sidebarTimer = null; syncSidebar(); }, 100);
  }
  // 延后监听：极少数情况下脚本执行时 body 尚未创建。
  const sidebarObserver = new MutationObserver(() => {
    if (!deletedIds.size || sidebarTimer) return;
    scheduleSidebar();
  });
  const cookie = key => {
    const value = document.cookie.split('; ').find(v => v.startsWith(key + '='));
    return value ? decodeURIComponent(value.slice(key.length + 1)) : '';
  };
  async function request(path, options = {}, timeoutMs = 30000) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), timeoutMs);
    try {
      const res = await fetch(path, {cache: 'no-store', ...options, credentials: 'same-origin', signal: controller.signal});
      if (!res.ok) throw new Error(res.status === 429 ? '请求过快，稍后重试' : `请求失败 (${res.status})`);
      if (res.status === 204) return null;
      const text = await res.text();
      if (!text) return null;
      let data;
      try { data = JSON.parse(text); } catch { throw new Error('响应异常，请刷新网页后重试'); }
      if (data?.success === false || data?.error) throw new Error('接口返回失败');
      return data;
    } finally { clearTimeout(timeout); }
  }
  async function session() {
    if (claude) {
      const orgs = await request('/api/organizations');
      if (!Array.isArray(orgs)) throw new Error('无法读取工作区');
      const active = cookie('lastActiveOrg');
      const org = orgs.find(o => o.uuid === active) || (orgs.length === 1 ? orgs[0] : null);
      if (!org?.uuid) throw new Error('请先在 Claude 选择工作区，再刷新列表');
      return {org: org.uuid, active};
    }
    const data = await request('/api/auth/session');
    if (!data?.accessToken) throw new Error('请先登录 ChatGPT');
    return {token: data.accessToken};
  }
  const headers = ctx => claude ? {} : {Authorization: `Bearer ${ctx.token}`};
  async function load(more = false) {
    if (busy) return;
    $('progress').hidden = true;
    busy = true; stopped = false;
    if (!more) { selected.clear(); items = []; context = null; nextOffset = 0; hasMore = false; }
    render();
    try {
      status('加载中…');
      context ||= await session();
      if (claude) {
        const data = await request(`/api/organizations/${encodeURIComponent(context.org)}/chat_conversations`);
        if (!Array.isArray(data)) throw new Error('对话列表格式已变化');
        items = data.map(x => ({id: x.uuid, title: x.name || '未命名对话'}));
      } else {
        const seen = new Set(items.map(x => x.id));
        {
          const data = await request(`/backend-api/conversations?offset=${nextOffset}&limit=100&order=updated`, {headers: headers(context)});
          if (!Array.isArray(data?.items)) throw new Error('对话列表格式已变化');
          let added = 0;
          for (const x of data.items) {
            if (!seen.has(x.id)) { seen.add(x.id); items.push({id: x.id, title: x.title || '未命名对话'}); added++; }
          }
          nextOffset += data.items.length;
          hasMore = data.items.length > 0 && (Number.isFinite(data.total) ? nextOffset < data.total : data.items.length === 100);
          status(`已加载 ${items.length} 条`);
          if (data.items.length && !added) { hasMore = false; throw new Error('分页异常，请刷新列表'); }
        }
      }
      items = items.filter(x => typeof x.id === 'string' && /^[a-z0-9_-]+$/i.test(x.id));
      // 服务器若仍返回某条记录，恢复显示，避免用隐藏掩盖删除失败。
      for (const item of items) deletedIds.delete(item.id);
      for (const node of document.querySelectorAll('[data-chat-cleaner-deleted]')) node.removeAttribute('data-chat-cleaner-deleted');
      syncSidebar();
      status(stopped ? '已停止加载' : `已加载 ${items.length} 条`);
    } catch (e) { status(e.message); }
    finally { busy = false; render(); }
  }
  async function removeSelected() {
    if (busy || !selected.size || !context) return;
    if (claude) return removeClaudeSelected();
    const queue = items.filter(x => selected.has(x.id));
    busy = true; stopped = false; render();
    let done = 0, cursor = 0, failure = null;
    const started = performance.now();
    const completed = new Set();
    $('progress').hidden = false;
    $('progress').max = queue.length;
    $('progress').value = 0;
    async function worker() {
      while (!stopped && !failure && cursor < queue.length) {
        const item = queue[cursor++];
        try {
          if (claude && cookie('lastActiveOrg') !== context.active) throw new Error('工作区已切换，请刷新列表');
          const id = encodeURIComponent(item.id);
          await request(claude
            ? `/api/organizations/${encodeURIComponent(context.org)}/chat_conversations/${id}`
            : `/backend-api/conversation/${id}`, {
            method: claude ? 'DELETE' : 'PATCH',
            headers: {...headers(context), 'Content-Type': 'application/json'},
            ...(claude ? {} : {body: JSON.stringify({is_visible: false})})
          });
          selected.delete(item.id);
          completed.add(item.id);
          deletedIds.add(item.id);
          syncSidebar();
          done++;
          $('progress').value = done;
          // 仅更新成功项。
          rowNodes.get(item.id)?.remove();
          rowNodes.delete(item.id);
          $('count').textContent = `${selected.size} / ${items.length - completed.size}`;
          status(`已删除 ${done}/${queue.length} · ${((performance.now() - started) / 1000).toFixed(1)}秒`);
        } catch (e) { failure ||= e; }
      }
    }
    try {
      status(`删除中 0/${queue.length}`);
      await Promise.all(Array.from({length: Math.min(4, queue.length)}, worker));
      status(failure
        ? `已删除 ${done} 条 · ${failure.message}`
        : `${stopped ? '已停止' : '完成'} · 已删除 ${done} 条 · ${((performance.now() - started) / 1000).toFixed(1)}秒`);
    } finally {
      items = items.filter(x => !completed.has(x.id));
      nextOffset = Math.max(0, nextOffset - completed.size);
      busy = false;
      render();
    }
  }

  async function removeClaudeSelected() {
    const queue = items.filter(x => selected.has(x.id));
    const org = context.org, active = context.active;
    const base = `/api/organizations/${encodeURIComponent(org)}/chat_conversations`;
    const batches = [];
    for (let i = 0; i < queue.length; i += CLAUDE_BATCH_SIZE) batches.push(queue.slice(i, i + CLAUDE_BATCH_SIZE));
    let cursor = 0, failure = null, submitted = 0, inFlight = 0;
    const attempted = new Set();
    const verified = new Set();
    const pendingBatches = new Set();
    let finished = false, checking = false;
    const started = performance.now();
    busy = true; stopped = false; render();
    const progress = () => {
      const elapsed = (performance.now() - started) / 1000;
      status(`已删除 ${verified.size}/${queue.length} · ${elapsed.toFixed(1)}秒`);
      $('progress').max = queue.length;
      $('progress').value = verified.size;
      $('progress').hidden = false;
    };
    const timer = setInterval(progress, 1000);
    // 删除接口可能长时间不结束；独立读取列表推进真实进度。
    async function verifyProgress() {
      if (finished || checking || !attempted.size) return;
      checking = true;
      try {
        if (cookie('lastActiveOrg') !== active) return;
        const rows = await request(base, {}, 15000);
        if (finished || !Array.isArray(rows)) return;
        const existing = new Set(rows.map(x => x.uuid));
        for (const id of attempted) {
          if (existing.has(id) || verified.has(id)) continue;
          verified.add(id); deletedIds.add(id); selected.delete(id);
          rowNodes.get(id)?.remove(); rowNodes.delete(id);
        }
        $('count').textContent = `${selected.size} / ${items.length - verified.size}`;
        scheduleSidebar(); progress();
        for (const pending of pendingBatches) {
          if (pending.ids.every(id => verified.has(id))) pending.resolve();
        }
      } catch { /* 单次核对失败不冒充成功；下一轮继续。 */ }
      finally { checking = false; }
    }
    const poller = setInterval(verifyProgress, 2000);
    async function worker() {
      while (!stopped && !failure && cursor < batches.length) {
        const batch = batches[cursor++];
        let sentAt = null;
        try {
          if (cookie('lastActiveOrg') !== active) throw new Error('工作区已切换，请刷新列表');
          batch.forEach(x => attempted.add(x.id));
          inFlight++;
          sentAt = performance.now();
          let resolveVerified;
          const verifiedBatch = new Promise(resolve => { resolveVerified = resolve; });
          const pending = {ids: batch.map(x => x.id), resolve: resolveVerified};
          pendingBatches.add(pending);
          const deletion = request(`${base}/delete_many`, {
            method: 'POST', headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({conversation_uuids: batch.map(x => x.id)})
          }, 600000);
          // 任一批的全部 ID 已从服务器列表消失，即可继续，不等悬挂响应。
          try { await Promise.race([deletion, verifiedBatch]); }
          finally { pendingBatches.delete(pending); }
          submitted += batch.length;
          await verifyProgress();
          progress();
        } catch (e) { failure ||= e; }
        finally { if (sentAt !== null) inFlight--; }
      }
    }
    try {
      progress();
      await Promise.all(Array.from({length: Math.min(CLAUDE_CONCURRENCY, batches.length)}, worker));
      clearInterval(timer);
      clearInterval(poller);
      if (cookie('lastActiveOrg') !== active) throw new Error('工作区已切换，请刷新列表');
      status('正在核对删除结果…');
      const remaining = verified.size === queue.length ? items.filter(x => !verified.has(x.id)).map(x => ({uuid: x.id, name: x.title})) : await request(base);
      if (!Array.isArray(remaining)) throw new Error('无法核对服务器列表，请刷新');
      const existing = new Set(remaining.map(x => x.uuid));
      let done = 0;
      for (const id of attempted) {
        if (!existing.has(id)) { done++; selected.delete(id); deletedIds.add(id); }
        else deletedIds.delete(id);
      }
      items = remaining.map(x => ({id: x.uuid, title: x.name || '未命名对话'}));
      for (const node of document.querySelectorAll('[data-chat-cleaner-deleted]')) node.removeAttribute('data-chat-cleaner-deleted');
      syncSidebar();
      const pending = [...attempted].filter(id => existing.has(id)).length;
      $('progress').value = done;
      status(`已核实删除 ${done} 条 · ${((performance.now() - started) / 1000).toFixed(1)}秒${pending ? ` · ${pending} 条仍在列表` : ''}${failure ? ` · ${failure.message}` : stopped ? ' · 已停止' : ''}`);
    } catch (e) {
      // 未核实的结果不算成功，也不持续隐藏。
      for (const id of attempted) if (!verified.has(id)) deletedIds.delete(id);
      items = items.filter(x => !verified.has(x.id));
      for (const node of document.querySelectorAll('[data-chat-cleaner-deleted]')) node.removeAttribute('data-chat-cleaner-deleted');
      syncSidebar();
      status(`结果未核实 · ${e.message}`);
    } finally { finished = true; clearInterval(timer); clearInterval(poller); busy = false; render(); }
  }

  const host = document.createElement('div');
  host.id = 'chat-cleaner-v2';
  // 不依赖 ChatGPT/Claude 侧边栏结构。入口永远固定在视口右下角。
  host.style.cssText =
    'position:fixed!important;right:16px!important;bottom:16px!important;' +
    'z-index:2147483647!important;display:flex!important;flex-direction:column;' +
    'align-items:flex-end;width:max-content;max-width:calc(100vw - 32px);' +
    'min-width:0;pointer-events:auto';
  const root = host.attachShadow({mode: 'open'});
  root.innerHTML = `
    <style>
      :host{font:13px/1.45 system-ui,-apple-system,"Segoe UI",sans-serif;min-width:0}
      *{box-sizing:border-box}[hidden]{display:none!important}
      button,input{font:inherit;color:inherit}button{cursor:pointer;background:transparent;border:0;border-radius:8px;padding:6px 8px}
      button:hover:not(:disabled),label:hover{background:var(--menu-hover,#8882)}button:disabled{opacity:.4;cursor:default}
      section{display:flex;flex-direction:column;width:min(360px,calc(100vw - 32px));max-height:min(75vh,680px);padding:12px;background:var(--cc-bg,#fff);color:var(--cc-text,#202124);border:1px solid var(--cc-border,#ddd);border-radius:14px;box-shadow:0 12px 40px #0004;margin-bottom:10px;overflow:hidden}
      header,footer,.tools{display:flex;align-items:center;gap:6px;flex:none}header{justify-content:space-between;padding:4px 8px 8px}header strong{font-size:13px;font-weight:600}header button{font-size:12px}
      #search{width:100%;min-width:0;height:34px;flex:none;background:transparent;border:1px solid #8884;border-radius:8px;padding:7px 9px;font-size:13px;outline-offset:2px}
      .tools{flex-wrap:wrap;padding:5px 0}.tools button{font-size:12px;padding:5px 6px}#count{margin-left:auto;font-size:11px;opacity:.6;white-space:nowrap}
      #list{overflow:auto;scrollbar-width:thin;min-height:50px;max-height:35vh;flex:0 1 auto}
      label{display:flex;align-items:center;gap:8px;padding:8px;border-radius:8px;cursor:pointer;font-size:13px}label span{min-width:0;flex:1;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
      input[type=checkbox]{width:15px;height:15px;margin:0;flex:none;accent-color:var(--menu-accent,currentColor)}
      footer{padding:8px 0 0;margin-top:5px;border-top:1px solid #8883}footer button{font-size:12px}#delete{flex:1;color:#d44b45}#delete:hover:not(:disabled){background:#d44b4515}
      #status{font-size:11px;opacity:.65;margin:7px 8px;overflow-wrap:anywhere}progress{width:100%;height:3px;flex:none;accent-color:var(--menu-accent,currentColor)}
      #toggle{display:inline-flex;align-items:center;justify-content:center;gap:8px;padding:10px 14px;border:1px solid var(--cc-border,#ddd);border-radius:999px;background:var(--cc-bg,#fff);color:var(--cc-text,#202124);box-shadow:0 5px 24px #0004;font-size:13px;font-weight:600}
      #toggle:hover:not(:disabled){background:var(--cc-hover,#f4f4f4)}
    </style>
    <section hidden>
      <header><strong>管理对话</strong><button id="close" aria-label="完成">完成</button></header>
      <input id="search" type="search" placeholder="搜索对话" aria-label="搜索对话">
      <div class="tools"><button id="refresh">刷新</button><button id="more" hidden>更多</button><button id="all">全选</button><button id="none">清空</button><span id="count"></span></div>
      <div id="list"></div>
      <footer><button id="delete">删除所选</button><button id="stop" disabled>停止</button></footer>
      <progress id="progress" hidden></progress>
      <div id="status" role="status">点击刷新加载</div>
    </section><button id="toggle"><svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.6" aria-hidden="true"><rect x="3" y="4" width="6" height="6" rx="1.5"/><path d="m4.5 7 1.2 1.2L8 5.8M13 7h8M13 17h8"/><rect x="3" y="14" width="6" height="6" rx="1.5"/></svg><span>管理对话</span></button>`;
  const $ = id => root.getElementById(id);
  const panel = root.querySelector('section');
  // 旧版必须先找到「新聊天」按钮才渲染 UI；ChatGPT 改版或侧栏收起时便彻底消失。
  // 新版直接挂载在 body，避免 React 重绘侧栏时删除脚本节点。
  function mountSidebar() {
    if (!document.body) return;
    if (!host.isConnected) document.body.appendChild(host);
    const dark = document.documentElement.classList.contains('dark') ||
      document.documentElement.getAttribute('data-theme') === 'dark';
    host.style.colorScheme = dark ? 'dark' : 'light';
    host.style.setProperty('--cc-bg', dark ? '#252525' : '#fff');
    host.style.setProperty('--cc-text', dark ? '#f4f4f4' : '#202124');
    host.style.setProperty('--cc-border', dark ? '#555' : '#dadce0');
    host.style.setProperty('--cc-hover', dark ? '#343434' : '#f3f4f6');
    host.style.setProperty('--menu-hover', dark ? '#ffffff12' : '#0000000a');
  }
  let uiStarted = false;
  function startUI() {
    if (uiStarted) return;
    uiStarted = true;
    sidebarObserver.observe(document.body, {childList: true, subtree: true});
    mountSidebar();
    window.addEventListener('resize', mountSidebar, {passive:true});
    // SPA 重新挂载页面节点后自动恢复入口。
    setInterval(() => { if (!document.hidden) mountSidebar(); }, 1500);
  }
  if (document.readyState === 'loading')
    document.addEventListener('DOMContentLoaded', startUI, {once:true});
  else startUI();
  const filtered = () => items.filter(x => String(x.title).toLocaleLowerCase().includes($('search').value.trim().toLocaleLowerCase()));
  function status(text) { $('status').textContent = text; }
  const rowNodes = new Map();
  function render() {
    rowNodes.clear();
    const scroll = $('list').scrollTop;
    const fragment = document.createDocumentFragment();
    for (const item of filtered()) {
      const row = document.createElement('label');
      rowNodes.set(item.id, row);
      const check = document.createElement('input');
      check.type = 'checkbox'; check.checked = selected.has(item.id); check.disabled = busy;
      check.onchange = () => {
        check.checked ? selected.add(item.id) : selected.delete(item.id);
        $('count').textContent = `${selected.size} / ${items.length}`;
        $('delete').disabled = busy || !selected.size;
        $('delete').textContent = selected.size ? `删除所选 (${selected.size})` : '删除所选';
      };
      const title = document.createElement('span');
      title.textContent = item.title; title.title = item.title;
      row.append(check, title); fragment.append(row);
    }
    $('list').replaceChildren(fragment); $('list').scrollTop = scroll;
    $('search').disabled = busy;
    $('count').textContent = `${selected.size} / ${items.length}`;
    $('more').hidden = !hasMore;
    for (const id of ['refresh', 'more', 'all', 'none']) $(id).disabled = busy;
    $('delete').disabled = busy || !selected.size;
    $('delete').textContent = selected.size ? `删除所选 (${selected.size})` : '删除所选';
    $('stop').disabled = !busy || stopped;
  }
  $('toggle').onclick = () => { panel.hidden = false; $('toggle').hidden = true; mountSidebar(); if (!context && !busy) load(); };
  $('close').onclick = () => { panel.hidden = true; $('toggle').hidden = false; mountSidebar(); };
  root.addEventListener('keydown', e => { if (e.key === 'Escape') { panel.hidden = true; $('toggle').hidden = false; mountSidebar(); $('toggle').focus(); } });
  $('refresh').onclick = () => load();
  $('more').onclick = () => load(true);
  $('search').oninput = render;
  $('all').onclick = () => { filtered().forEach(x => selected.add(x.id)); render(); };
  $('none').onclick = () => { selected.clear(); render(); };
  $('delete').onclick = removeSelected;
  $('stop').onclick = () => { stopped = true; status('正在停止…'); render(); };
  render();
})();
