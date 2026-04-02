<script>
  const SNAP = 15;
  const PX_PER_MIN = 0.5;
  const snap = (min) => Math.round(min / SNAP) * SNAP;
  const pxToMin = (px) => px / PX_PER_MIN;
  const minToPx = (min) => min * PX_PER_MIN;
  const format = (min) => {
    const h = Math.floor(min / 60);
    const m = min % 60;
    return `${h}:${m.toString().padStart(2, '0')}`;
  };
  const formatDuration = (min) => {
    const h = Math.floor(min / 60);
    const m = min % 60;
    if (h === 0) return `${m}m`;
    if (m === 0) return `${h}h`;
    return `${h}h${m}m`;
  };

  const DAY_START = 180;
  const DAY_END = 1620;
  const COLS = [0, 1, 2];
  const hours = Array.from({ length: 25 }, (_, i) => i + 3);
  const gridLines = Array.from({ length: 24 * 4 }, (_, i) => DAY_START + i * 15);

  // 列ごとに独立した重なり検出
  const layoutTasks = $derived(() => {
    const result = [];
    for (const side of cols) {
      const sideTasks = [...tasks].filter(t => t.col === side).sort((a, b) => a.start - b.start);
      const cols = [];
      const withSubCol = sideTasks.map(task => {
        let subCol = cols.findIndex(end => end <= task.start);
        if (subCol === -1) subCol = cols.length;
        cols[subCol] = task.end;
        return { ...task, subCol };
      });
      const final = withSubCol.map(task => {
        const overlapping = withSubCol.filter(t => t.start < task.end && t.end > task.start);
        const totalSubCols = Math.max(...overlapping.map(t => t.subCol)) + 1;
        return { ...task, totalSubCols };
      });
      result.push(...final);
    }
    return result;
  });

  let stock = $state([]);
  let tasks = $state([
    { id:  1, title: '睡眠',   start:  180, end:  480, col: 0 },
    { id:  2, title: '朝食',   start:  480, end:  540, col: 0 },
    { id:  3, title: '開発',   start:  540, end:  780, col: 0 },
    { id:  4, title: '1コマ',  start:  540, end:  630, col: 1 },
    { id:  5, title: 'MTG',    start:  630, end:  660, col: 1 },
    { id:  6, title: '2コマ',  start:  690, end:  780, col: 1 },
    { id:  7, title: '昼食',   start:  780, end:  840, col: 0 },
    { id:  8, title: '開発',   start:  840, end: 1080, col: 0 },
    { id:  9, title: '3コマ',  start:  840, end:  930, col: 1 },
    { id: 10, title: 'MTG',    start:  930, end:  960, col: 1 },
    { id: 11, title: '4コマ',  start:  990, end: 1080, col: 1 },
    { id: 12, title: '料理',   start: 1080, end: 1140, col: 0 },
    { id: 13, title: '夕食',   start: 1140, end: 1200, col: 0 },
    { id: 14, title: 'お風呂', start: 1200, end: 1260, col: 0 },
    { id: 15, title: '5コマ',  start: 1260, end: 1350, col: 1 },
    { id: 16, title: '睡眠',   start: 1380, end: 1620, col: 0 },
  ]);
  let nextId = $state(17);
  let newTitle = $state('');

  let dragging = $state(null);
  let container = $state(null);
  let cols = $state([...COLS]); // 動的に増減する列IDリスト
  let colRefs = $state(COLS.map(() => null));
  let stockContainer = $state(null);
  let stockDrag = $state(null);
  let tooltip = $state(null);

  // 列幅リサイズ
  let colWidths = $state(COLS.map(() => 1)); // flex値
  let resizing = $state(null); // { dividerIndex, startX, leftWidth, rightWidth }
  let wrapperRef = $state(null);

  function addCol() {
    const newColId = Math.max(...cols) + 1;
    cols = [...cols, newColId];
    colRefs = [...colRefs, null];
    colWidths = [...colWidths, 1];
  }

  function removeCol() {
    if (cols.length <= 1) return;
    const removedCol = cols[cols.length - 1];
    // 削除列のタスクをストックへ
    const removed = tasks.filter(t => t.col === removedCol);
    stock = [...stock, ...removed.map(t => ({ id: t.id, title: t.title }))];
    tasks = tasks.filter(t => t.col !== removedCol);
    cols = cols.slice(0, -1);
    colRefs = colRefs.slice(0, -1);
    colWidths = colWidths.slice(0, -1);
  }

  function onDividerMousedown(e, dividerIndex) {
    e.preventDefault();
    resizing = {
      dividerIndex,
      startX: e.clientX,
      leftWidth: colWidths[dividerIndex],
      rightWidth: colWidths[dividerIndex + 1],
    };
  }

  const getNowMin = () => {
    const now = new Date();
    return now.getHours() * 60 + now.getMinutes();
  };
  let nowMin = $state(getNowMin());
  setInterval(() => { nowMin = getNowMin(); }, 60000);

  // タイムライン内ドラッグ
  function onTaskMousedown(e, task) {
    e.preventDefault();
    tooltip = null;
    const colEl = colRefs[task.col];
    const rect = colEl.getBoundingClientRect();
    const y = e.clientY - rect.top;
    dragging = { id: task.id, type: 'move', col: task.col, offsetY: y - minToPx(task.start - DAY_START) };
  }

  function onResizeMousedown(e, task) {
    e.preventDefault();
    e.stopPropagation();
    tooltip = null;
    dragging = { id: task.id, type: 'resize', col: task.col };
  }

  function onStockMousedown(e, item) {
    e.preventDefault();
    stockDrag = { id: item.id, title: item.title, ghostX: e.clientX, ghostY: e.clientY };
  }

  function returnToStock(e, task) {
    e.stopPropagation();
    tooltip = null;
    tasks = tasks.filter(t => t.id !== task.id);
    stock = [...stock, { id: task.id, title: task.title }];
  }

  function onMousemove(e) {
    if (resizing) {
      const totalWidth = colWidths.reduce((a, b) => a + b, 0);
      const wrapperRect = wrapperRef.getBoundingClientRect();
      // hour-labels(48px) + dividers(1px each) を除いた実幅
      const availableWidth = wrapperRect.width - 48 - (COLS.length - 1);
      const dx = e.clientX - resizing.startX;
      const dFlex = dx / availableWidth * totalWidth;
      const newLeft = Math.max(0.1, resizing.leftWidth + dFlex);
      const newRight = Math.max(0.1, resizing.rightWidth - dFlex);
      const updated = [...colWidths];
      updated[resizing.dividerIndex] = newLeft;
      updated[resizing.dividerIndex + 1] = newRight;
      colWidths = updated;
      return;
    }
    if (stockDrag) {
      stockDrag = { ...stockDrag, ghostX: e.clientX, ghostY: e.clientY };
      return;
    }
    if (!dragging) return;
    const colEl = colRefs[dragging.col];
    const rect = colEl.getBoundingClientRect();
    const y = e.clientY - rect.top;

    tasks = tasks.map((t) => {
      if (t.id !== dragging.id) return t;
      if (dragging.type === 'move') {
        const newStart = snap(Math.max(DAY_START, DAY_START + pxToMin(y - dragging.offsetY)));
        const duration = t.end - t.start;
        const newEnd = newStart + duration;
        if (newEnd > DAY_END) return t;
        return { ...t, start: newStart, end: newEnd };
      }
      if (dragging.type === 'resize') {
        const newEnd = snap(DAY_START + pxToMin(y));
        if (newEnd < t.start + 30) return t;
        if (newEnd > DAY_END) return t;
        return { ...t, end: newEnd };
      }
      return t;
    });
  }

  function onMouseup(e) {
    if (resizing) {
      resizing = null;
      return;
    }
    if (stockDrag) {
      const x = e.clientX;
      const y = e.clientY;
      let dropped = false;

      for (const side of cols) {
        const colEl = colRefs[side];
        if (!colEl) continue;
        const rect = colEl.getBoundingClientRect();
        if (x >= rect.left && x <= rect.right && y >= rect.top && y <= rect.bottom) {
          const relY = y - rect.top;
          const start = snap(Math.max(DAY_START, DAY_START + pxToMin(relY - minToPx(30))));
          const end = Math.min(start + 60, DAY_END);
          tasks = [...tasks, { id: stockDrag.id, title: stockDrag.title, col: side, start, end }];
          stock = stock.filter(s => s.id !== stockDrag.id);
          dropped = true;
          break;
        }
      }

      if (!dropped && !stock.find(s => s.id === stockDrag.id)) {
        stock = [...stock, { id: stockDrag.id, title: stockDrag.title }];
      }
      stockDrag = null;
      return;
    }

    // タイムライン内ドラッグ終了時、列をまたいでいたら col を更新
    if (dragging && dragging.type === 'move') {
      const x = e.clientX;
      for (const side of cols) {
        const colEl = colRefs[side];
        if (!colEl) continue;
        const rect = colEl.getBoundingClientRect();
        if (x >= rect.left && x <= rect.right) {
          tasks = tasks.map(t => t.id === dragging.id ? { ...t, col: side } : t);
          break;
        }
      }
    }
    dragging = null;
  }

  // ツールチップ
  function onTaskMouseenter(e, task) {
    if (dragging || stockDrag) return;
    const colEl = colRefs[task.col];
    const rect = colEl.getBoundingClientRect();
    tooltip = {
      text: `${task.title}　${format(task.start)} – ${format(task.end)} (${formatDuration(task.end - task.start)})`,
      x: e.clientX - rect.left + 12,
      y: e.clientY - rect.top - 10,
      col: task.col,
    };
  }

  function onTaskMousemove(e, task) {
    if (!tooltip || dragging || stockDrag) return;
    const colEl = colRefs[task.col];
    const rect = colEl.getBoundingClientRect();
    tooltip = { ...tooltip, x: e.clientX - rect.left + 12, y: e.clientY - rect.top - 10 };
  }

  function onTaskMouseleave() { tooltip = null; }

  function addTask() {
    const title = newTitle.trim();
    if (!title) return;
    stock = [...stock, { id: nextId++, title }];
    newTitle = '';
  }

  function onFormKeydown(e) {
    if (e.key === 'Enter') addTask();
  }

  let isDraggingAny = $derived(!!(dragging || stockDrag || resizing));
</script>

<svelte:window onmousemove={onMousemove} onmouseup={onMouseup} />

{#if stockDrag}
  <div class="ghost" style="left: {stockDrag.ghostX + 8}px; top: {stockDrag.ghostY - 16}px;">
    {stockDrag.title}
  </div>
{/if}

<div class="app" class:no-select={isDraggingAny}>
  <div class="header">
    <h1>1日タスクタイムライン</h1>
    <div class="col-btns">
      <button class="col-btn" onclick={removeCol} disabled={cols.length <= 1}>－ 列</button>
      <button class="col-btn" onclick={addCol}>＋ 列</button>
    </div>
  </div>
  <div class="layout">

    <!-- ストック -->
    <div class="stock-panel" bind:this={stockContainer}>
      <div class="stock-label">ストック</div>
      <div class="stock-list">
        {#each stock as item (item.id)}
          <div class="stock-item" onmousedown={(e) => onStockMousedown(e, item)}>
            {item.title}
          </div>
        {/each}
        {#if stock.length === 0}
          <div class="stock-empty">空</div>
        {/if}
      </div>
    </div>

    <!-- タイムライン（左右2列） -->
    <div class="wrapper" bind:this={wrapperRef}>
      <div class="hour-labels">
        {#each hours as h}
          <div class="hour-label" style="top: {minToPx(h * 60 - DAY_START)}px">{h}:00</div>
        {/each}
      </div>

      {#each cols as col, i}
        {#if i > 0}
          <div
            class="col-divider"
            class:resizing={resizing?.dividerIndex === i - 1}
            onmousedown={(e) => onDividerMousedown(e, i - 1)}
          ></div>
        {/if}
        <div class="timeline-col" style="flex: {colWidths[col]}" bind:this={colRefs[col]}>
          {#each gridLines as min}
            <div class="grid-line" class:major={min % 60 === 0} style="top: {minToPx(min - DAY_START)}px"></div>
          {/each}
          <div class="night-overlay" style="top: 0; height: {minToPx(420 - DAY_START)}px;"></div>
          <div class="night-overlay" style="top: {minToPx(1380 - DAY_START)}px;"></div>
          {#if nowMin >= DAY_START && nowMin <= DAY_END}
            <div class="now-line" style="top: {minToPx(nowMin - DAY_START)}px"></div>
          {/if}
          {#each layoutTasks().filter(t => t.col === col) as task (task.id)}
            <div
              class="task"
              style="top: {minToPx(task.start - DAY_START)}px; height: calc({minToPx(task.end - task.start)}px - 3px); left: calc(4px + {task.subCol / task.totalSubCols * 100}%); right: calc(4px + {(task.totalSubCols - task.subCol - 1) / task.totalSubCols * 100}%);"
              data-size={task.end - task.start <= 30 ? 'xs' : task.end - task.start <= 45 ? 'sm' : 'md'}
              onmousedown={(e) => onTaskMousedown(e, task)}
              onmouseenter={(e) => onTaskMouseenter(e, task)}
              onmousemove={(e) => onTaskMousemove(e, task)}
              onmouseleave={onTaskMouseleave}
            >
              <div class="task-title">{task.title}</div>
              <div class="task-time">{format(task.start)} – {format(task.end)} ({formatDuration(task.end - task.start)})</div>
              <div class="task-actions">
                {#each [30, 60, 90] as min}
                  <button class="dur-btn" class:active={task.end - task.start === min}
                    onmousedown={(e) => { e.stopPropagation(); tasks = tasks.map(t => t.id === task.id ? { ...t, end: Math.min(t.start + min, DAY_END) } : t); }}
                  >{min}</button>
                {/each}
                <button class="return-btn" onmousedown={(e) => returnToStock(e, task)}>×</button>
              </div>
              <div class="resize-handle" onmousedown={(e) => onResizeMousedown(e, task)}></div>
            </div>
          {/each}
          {#if tooltip && tooltip.col === col}
            <div class="tooltip" style="left: {tooltip.x}px; top: {tooltip.y}px;">{tooltip.text}</div>
          {/if}
        </div>
      {/each}
    </div>
  </div>

  <div class="add-form">
    <input type="text" placeholder="タスク名を入力" bind:value={newTitle} onkeydown={onFormKeydown} />
    <button class="add-btn" onclick={addTask}>追加</button>
  </div>
</div>

<style>
  * { box-sizing: border-box; margin: 0; padding: 0; }

  .app {
    font-family: sans-serif;
    padding: 24px;
    background: #F4EBDA;
    min-height: 100vh;
  }

  .no-select { user-select: none; }

  .header {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;
  }

  h1 {
    font-size: 18px;
    font-weight: 600;
    color: #262724;
  }

  .col-btns {
    display: flex;
    gap: 4px;
  }

  .col-btn {
    padding: 4px 10px;
    font-size: 12px;
    font-weight: 600;
    background: #fdf8f2;
    color: #262724;
    border: 1px solid #c8bfb0;
    border-radius: 5px;
    cursor: pointer;
  }

  .col-btn:hover { background: #ede5d8; }
  .col-btn:disabled { opacity: 0.35; cursor: default; }

  .layout {
    display: flex;
    gap: 12px;
    align-items: flex-start;
  }

  /* ストック */
  .stock-panel {
    width: 120px;
    flex-shrink: 0;
    border: 1px solid #c8bfb0;
    border-radius: 8px;
    background: #fdf8f2;
    overflow: hidden;
  }

  .stock-label {
    font-size: 10px;
    font-weight: 600;
    color: #8a8278;
    padding: 7px 10px 5px;
    border-bottom: 1px solid #e0d8cc;
    letter-spacing: 0.05em;
    text-transform: uppercase;
  }

  .stock-list {
    padding: 6px;
    display: flex;
    flex-direction: column;
    gap: 4px;
    min-height: 48px;
  }

  .stock-item {
    background: #A6B5A5;
    color: #262724;
    font-size: 12px;
    font-weight: 600;
    padding: 5px 8px;
    border-radius: 5px;
    cursor: grab;
    box-shadow: 0 1px 3px rgba(38, 39, 36, 0.1);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .stock-item:active { cursor: grabbing; }

  .stock-empty {
    font-size: 10px;
    color: #c0b8b0;
    text-align: center;
    padding: 10px 4px;
  }

  /* ゴースト */
  .ghost {
    position: fixed;
    background: #A6B5A5;
    color: #262724;
    font-size: 13px;
    font-weight: 600;
    padding: 5px 10px;
    border-radius: 5px;
    box-shadow: 0 4px 12px rgba(38, 39, 36, 0.2);
    pointer-events: none;
    z-index: 999;
    opacity: 0.85;
  }

  /* タイムライン */
  .wrapper {
    flex: 1;
    display: flex;
    overflow-y: auto;
    max-height: calc(100vh - 80px);
    border: 1px solid #c8bfb0;
    border-radius: 8px;
    background: #fdf8f2;
  }

  .hour-labels {
    position: relative;
    width: 48px;
    height: 720px;
    flex-shrink: 0;
    background: #f4ede0;
    border-right: 1px solid #d9d0c4;
  }

  .hour-labels::before {
    content: '';
    position: absolute;
    left: 0; right: 0; top: 0;
    height: 120px;
    background: rgba(38, 39, 36, 0.06);
    pointer-events: none;
  }

  .hour-labels::after {
    content: '';
    position: absolute;
    left: 0; right: 0; top: 600px; bottom: 0;
    background: rgba(38, 39, 36, 0.06);
    pointer-events: none;
  }

  .hour-label {
    position: absolute;
    right: 6px;
    font-size: 11px;
    color: #8a8278;
    transform: translateY(-50%);
    user-select: none;
  }

  .timeline-col {
    position: relative;
    flex: 1;
    height: 720px;
    background: #fdf8f2;
    user-select: none;
  }

  .col-divider {
    width: 5px;
    height: 720px;
    background: #d9d0c4;
    flex-shrink: 0;
    cursor: col-resize;
    transition: background 0.15s;
  }

  .col-divider:hover, .col-divider.resizing {
    background: #A6B5A5;
  }

  .night-overlay {
    position: absolute;
    left: 0; right: 0; bottom: 0;
    background: rgba(38, 39, 36, 0.06);
    pointer-events: none;
    z-index: 1;
  }

  .night-overlay[style*="height"] { bottom: unset; }

  .grid-line {
    position: absolute;
    left: 0; right: 0;
    height: 1px;
    background: #ede5d8;
    pointer-events: none;
  }

  .grid-line.major { background: #d9d0c4; }

  .now-line {
    position: absolute;
    left: 0; right: 0;
    height: 2px;
    background: #b06a50;
    pointer-events: none;
    z-index: 10;
  }

  .now-line::before {
    content: '';
    position: absolute;
    left: -4px; top: -4px;
    width: 10px; height: 10px;
    border-radius: 50%;
    background: #b06a50;
  }

  .task {
    position: absolute;
    left: 4px; right: 4px;
    background: #A6B5A5;
    border-radius: 6px;
    padding: 6px 8px;
    cursor: grab;
    color: #262724;
    overflow: hidden;
    box-shadow: 0 1px 4px rgba(38, 39, 36, 0.12);
    border-bottom: 1px solid rgba(38, 39, 36, 0.2);
  }

  .task[data-size='xs'] { padding: 2px 6px; }
  .task[data-size='xs'] .task-title { font-size: 11px; line-height: 1.2; }
  .task[data-size='xs'] .task-time { display: none; }
  .task[data-size='sm'] { padding: 3px 7px; }
  .task[data-size='sm'] .task-title { font-size: 12px; }
  .task[data-size='sm'] .task-time { font-size: 10px; margin-top: 1px; }

  .task:active { cursor: grabbing; }

  .task-actions {
    position: absolute;
    top: 3px; right: 4px;
    display: flex;
    gap: 2px;
    align-items: center;
    opacity: 0;
    transition: opacity 0.15s;
  }

  .task:hover .task-actions { opacity: 1; }

  .dur-btn, .return-btn {
    background: rgba(38, 39, 36, 0.1);
    border: none;
    color: #262724;
    font-size: 11px;
    font-weight: 600;
    line-height: 1;
    padding: 2px 4px;
    cursor: pointer;
    border-radius: 3px;
  }

  .dur-btn:hover, .return-btn:hover { background: rgba(38, 39, 36, 0.25); }
  .dur-btn.active { background: rgba(38, 39, 36, 0.3); }

  .task-title {
    font-size: 13px;
    font-weight: 600;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .task-time { font-size: 11px; opacity: 0.7; margin-top: 2px; }

  .resize-handle {
    position: absolute;
    bottom: 0; left: 0; right: 0;
    height: 8px;
    cursor: ns-resize;
    background: rgba(38, 39, 36, 0.12);
    border-radius: 0 0 6px 6px;
    transition: right 0.15s;
  }

  .task:hover .resize-handle { right: 108px; border-radius: 0 0 0 6px; }
  .resize-handle:hover { background: rgba(38, 39, 36, 0.25); }

  .tooltip {
    position: absolute;
    background: #262724;
    color: #F4EBDA;
    font-size: 12px;
    white-space: nowrap;
    padding: 4px 8px;
    border-radius: 4px;
    pointer-events: none;
    z-index: 200;
  }

  .add-form {
    display: flex;
    gap: 8px;
    margin-top: 12px;
    align-items: center;
  }

  .add-form input {
    flex: 1;
    padding: 8px 12px;
    font-size: 14px;
    border: 1px solid #c8bfb0;
    border-radius: 6px;
    outline: none;
    background: #fdf8f2;
    color: #262724;
  }

  .add-form input:focus { border-color: #A6B5A5; }

  .add-btn {
    padding: 8px 16px;
    font-size: 14px;
    background: #A6B5A5;
    color: #262724;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-weight: 600;
  }

  .add-btn:hover { background: #8fa48e; }
</style>
