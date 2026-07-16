# rg.github.io
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RG 壓延/研磨智慧生產調度與庫存看板</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+TC:wght=300;400;700&display=swap');
        body { font-family: 'Noto Sans TC', sans-serif; background-color: #f1f5f9; color: #0f172a; }
        .urgent-pulse { animation: pulse 1.5s infinite; }
        @keyframes pulse { 0% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.5); } 70% { box-shadow: 0 0 0 8px rgba(239, 68, 68, 0); } 100% { box-shadow: 0 0 0 0 rgba(239, 68, 68, 0); } }
    </style>
</head>
<body class="min-h-screen p-4 lg:p-6">

    <!-- 頂部頁眉區 -->
    <header class="flex flex-col xl:flex-row justify-between items-start xl:items-center mb-6 bg-white p-6 rounded-2xl border border-slate-200 shadow-xl gap-4">
        <div>
            <div class="flex items-center gap-3">
                <h1 class="text-2xl lg:text-3xl font-bold bg-gradient-to-r from-cyan-600 to-emerald-600 bg-clip-text text-transparent">
                    RG 壓延/研磨智慧調度與預查看板
                </h1>
                <span id="role-badge" class="px-2.5 py-0.5 rounded-full text-xs font-bold bg-emerald-100 text-emerald-700 border border-emerald-200">
                    <i class="fa-solid fa-user-shield mr-1"></i>作業長權限
                </span>
            </div>
            <div class="flex flex-wrap gap-4 mt-2 text-slate-500 text-sm">
                <span><i class="fa-solid fa-clock text-cyan-600"></i> 目前班別：<b id="current-shift" class="text-emerald-600">A班 (早班)</b></span>
                <span><i class="fa-solid fa-calendar-day"></i> <span id="current-time" class="font-mono font-bold text-slate-700"></span></span>
                <span class="text-slate-400">| 切換身分：
                    <select id="role-selector" onchange="switchRole(this.value)" class="bg-slate-100 border border-slate-300 rounded px-1.5 py-0.5 text-xs text-slate-700 font-bold focus:outline-none">
                        <option value="leader">作業長 (全權限)</option>
                        <option value="operator">一般作業員 (唯讀)</option>
                    </select>
                </span>
            </div>
        </div>
        
        <!-- 生產排程指派面板 -->
        <div id="admin-dispatch-panel" class="bg-slate-50 p-4 rounded-xl border border-slate-200 flex flex-wrap gap-3 items-end w-full xl:w-auto shadow-inner">
            <div class="w-24">
                <label class="block text-xs text-slate-500 mb-1 font-bold">派單日期</label>
                <select id="ins-date" class="w-full bg-white border border-slate-300 rounded p-1 text-slate-700 text-sm font-bold focus:outline-none">
                    <option value="today">今日工單</option>
                    <option value="tomorrow">明日預排</option>
                </select>
            </div>
            <div class="w-16">
                <label class="block text-xs text-slate-500 mb-1 font-bold">預設順序</label>
                <input type="number" id="ins-seq" class="w-full bg-white border border-slate-300 rounded p-1 text-center font-bold text-cyan-600" value="1">
            </div>
            <div class="w-24">
                <label class="block text-xs text-slate-500 mb-1 font-bold">對應機台</label>
                <select id="ins-machine" onchange="adaptSpecOptions()" class="w-full bg-white border border-slate-300 rounded p-1 text-slate-700 text-sm font-medium focus:outline-none">
                    <option value="ZH1">ZH1</option>
                    <option value="TL">TL</option>
                    <option value="SP">SP</option>
                </select>
            </div>
            <div class="w-44">
                <label class="block text-xs text-slate-500 mb-1 font-bold">輪子細分規格</label>
                <select id="ins-spec" class="w-full bg-white border border-slate-300 rounded p-1 text-slate-700 text-sm font-medium focus:outline-none">
                </select>
            </div>
            <div class="w-20">
                <label class="block text-xs text-slate-500 mb-1 font-bold">製程番號</label>
                <input type="text" id="ins-process" class="w-full bg-white border border-slate-300 rounded p-1 text-slate-700 text-center text-sm font-medium" value="#60">
            </div>
            <div class="w-28">
                <label class="block text-xs text-slate-500 mb-1 font-bold">預計完成 (手打24H)</label>
                <input type="text" id="ins-etf" class="w-full bg-white border border-slate-300 rounded p-1 text-emerald-600 font-mono text-sm font-extrabold text-center" value="15:30" placeholder="例如 08:00">
            </div>
            
            <div class="flex gap-2">
                <button onclick="insertWorkOrder(false)" class="bg-emerald-600 hover:bg-emerald-500 text-white px-3 py-1.5 rounded-lg font-bold transition-all text-sm flex items-center gap-1.5 cursor-pointer shadow-md">
                    <i class="fa-solid fa-list-check"></i> 新增派單
                </button>
                <button onclick="insertWorkOrder(true)" class="bg-red-600 hover:bg-red-500 text-white px-3 py-1.5 rounded-lg font-bold transition-all text-sm flex items-center gap-1.5 cursor-pointer shadow-md">
                    <i class="fa-solid fa-bolt"></i> 緊急插單
                </button>
            </div>
        </div>
    </header>

    <!-- 數據分析看塊 -->
    <section class="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
        <div class="bg-white p-5 rounded-2xl border border-slate-200 shadow-sm flex items-center justify-between">
            <div>
                <p class="text-xs lg:text-sm font-bold text-slate-400 uppercase tracking-wider"><span class="target-date-label">今日</span>總工單量</p>
                <p class="text-2xl lg:text-3xl font-extrabold text-slate-800 mt-1" id="dash-total">0 組</p>
            </div>
            <div class="w-12 h-12 rounded-xl bg-slate-100 flex items-center justify-center text-slate-500 text-xl"><i class="fa-solid fa-folder-open"></i></div>
        </div>
        <div class="bg-white p-5 rounded-2xl border border-slate-200 shadow-sm flex items-center justify-between border-l-4 border-l-slate-400">
            <div>
                <p class="text-xs lg:text-sm font-bold text-slate-400 uppercase tracking-wider">排隊待研磨</p>
                <p class="text-2xl lg:text-3xl font-extrabold text-slate-700 mt-1" id="dash-waiting">0 組</p>
            </div>
            <div class="w-12 h-12 rounded-xl bg-slate-50 flex items-center justify-center text-slate-400 text-xl"><i class="fa-solid fa-hourglass-start"></i></div>
        </div>
        <div class="bg-white p-5 rounded-2xl border border-slate-200 shadow-sm flex items-center justify-between border-l-4 border-l-cyan-500 transition-all">
            <div class="max-w-[80%]">
                <p class="text-xs lg:text-sm font-bold text-cyan-600 uppercase tracking-wider">現場正在研磨</p>
                <p class="text-2xl lg:text-3xl font-extrabold text-cyan-600 mt-1" id="dash-processing">0 組</p>
                <p class="text-xs text-cyan-600 mt-1 font-bold truncate max-w-full" id="dash-processing-names">暫無研磨任務</p>
            </div>
            <div class="w-12 h-12 rounded-xl bg-cyan-50 flex items-center justify-center text-cyan-500 text-xl"><i class="fa-solid fa-gear fa-spin"></i></div>
        </div>
        <div class="bg-white p-5 rounded-2xl border border-slate-200 shadow-sm flex items-center justify-between border-l-4 border-l-emerald-500">
            <div>
                <p class="text-xs lg:text-sm font-bold text-emerald-600 uppercase tracking-wider">累計已完工</p>
                <p class="text-2xl lg:text-3xl font-extrabold text-emerald-600 mt-1" id="dash-completed">0 組</p>
            </div>
            <div class="w-12 h-12 rounded-xl bg-emerald-50 flex items-center justify-center text-emerald-500 text-xl"><i class="fa-solid fa-circle-check"></i></div>
        </div>
    </section>

    <!-- 主篩選工具列 -->
    <section class="flex flex-col lg:flex-row justify-between items-start lg:items-center gap-4 mb-6">
        <div class="flex flex-wrap items-center gap-3 w-full lg:w-auto">
            <div class="bg-slate-200 p-1 rounded-xl flex gap-1 shadow-sm">
                <button onclick="switchDateView('today')" id="tab-date-today" class="px-5 py-2 text-sm font-bold rounded-lg bg-white text-slate-800 shadow transition-all cursor-pointer flex items-center gap-2">
                    <i class="fa-solid fa-calendar-day text-emerald-600"></i> 今日動態排程
                </button>
                <button onclick="switchDateView('tomorrow')" id="tab-date-tomorrow" class="px-5 py-2 text-sm font-bold rounded-lg text-slate-600 hover:bg-slate-100/60 transition-all cursor-pointer flex items-center gap-2">
                    <i class="fa-solid fa-calendar-plus text-amber-600"></i> 明日預排工單
                </button>
            </div>
            
            <button id="btn-move-tomorrow-to-today" onclick="moveTomorrowToToday()" class="bg-gradient-to-r from-amber-500 to-orange-600 hover:from-amber-600 hover:to-orange-700 text-white px-4 py-2 rounded-xl text-sm font-bold shadow-md transition-all flex items-center gap-2 cursor-pointer border border-amber-600">
                <i class="fa-solid fa-angles-left"></i> 一鍵將明日工單移至今日
            </button>
        </div>

        <div class="flex flex-col md:flex-row gap-4 w-full lg:w-auto items-center">
            <div class="bg-white p-1.5 rounded-xl border border-slate-200 flex gap-1 w-full md:w-auto shadow-sm">
                <button onclick="filterStatus('全部')" id="btn-filter-全部" class="px-3.5 py-1 text-xs font-bold rounded-md bg-cyan-600 text-white transition-all cursor-pointer">全部</button>
                <button onclick="filterStatus('待研磨')" id="btn-filter-待研磨" class="px-3.5 py-1 text-xs font-bold rounded-md text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">待研磨</button>
                <button onclick="filterStatus('進行中')" id="btn-filter-進行中" class="px-3.5 py-1 text-xs font-bold rounded-md text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">研磨中</button>
                <button onclick="filterStatus('已完工')" id="btn-filter-已完工" class="px-3.5 py-1 text-xs font-bold rounded-md text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">已完工</button>
            </div>
            <button onclick="toggleHistoryModal(true)" class="w-full md:w-auto bg-slate-200 hover:bg-slate-300 border border-slate-300 text-slate-700 px-4 py-2 rounded-xl text-xs font-bold transition-all flex items-center justify-center gap-1.5 cursor-pointer shadow-sm">
                <i class="fa-solid fa-clock-rotate-left text-amber-600"></i> 追查歷史完工資料
            </button>
        </div>
    </section>

    <!-- 生產排程主看板 -->
    <main class="bg-white rounded-2xl shadow-xl overflow-hidden border border-slate-200">
        <div class="p-4 bg-slate-50 border-b border-slate-200 flex justify-between items-center">
            <span class="text-sm font-bold text-slate-500">當前正在檢視：<span id="current-date-view-title" class="text-emerald-600 font-extrabold text-base">今日動態排程</span> (<span id="current-filter-label" class="text-cyan-600">全部</span>)</span>
            <span id="role-hint-message" class="text-xs text-slate-400 font-medium">提示：時間已全面換裝為手動鍵入模式，可自由輸入 24H 制數值（如 06:15、23:40）</span>
        </div>
        <div class="overflow-x-auto">
            <table class="w-full text-left table-fixed min-w-[1200px]">
                <thead>
                    <tr class="bg-slate-100 text-slate-600 text-sm font-bold tracking-wider border-b border-slate-200">
                        <th class="p-4 w-16 text-center">順序</th>
                        <th class="p-4 w-20 text-center">機台</th>
                        <th class="p-4 w-52">輪子細分規格</th>
                        <th class="p-4 w-16">番號</th>
                        <th class="p-4 w-24">指派班別</th>
                        <th class="p-4 w-52">工藝用途與備註說明</th>
                        <th class="p-4 w-44 text-emerald-700 font-bold"><i class="fa-regular fa-clock mr-1"></i>預計完成時間 (手打24H)</th>
                        <th class="p-4 w-44 action-col">作業狀態更新</th>
                        <th class="p-4 w-24 text-center action-col">順序調整</th>
                        <th class="p-4 w-22 text-center text-red-600 action-col">排單刪除</th>
                        <th class="p-4 w-20 text-center action-col">完工結案</th>
                    </tr>
                </thead>
                <tbody id="kanban-table-body" class="divide-y divide-slate-100">
                </tbody>
            </table>
        </div>
    </main>

    <!-- 📦 全部庫存與待磨毛胚管理區 (新增功能) -->
    <section class="mt-8 bg-white rounded-2xl shadow-xl overflow-hidden border border-slate-200">
        <div class="p-5 bg-slate-900 border-b border-slate-800 flex flex-col xl:flex-row justify-between items-start xl:items-center gap-4">
            <div class="flex items-center gap-3">
                <div class="w-10 h-10 rounded-xl bg-cyan-500/10 flex items-center justify-center text-cyan-400 text-lg">
                    <i class="fa-solid fa-warehouse"></i>
                </div>
                <div>
                    <h3 class="text-lg font-bold text-white">📦 廠內工作輪全部庫存表</h3>
                    <p class="text-xs text-slate-400 mt-0.5">追蹤廠內所有工作輪/中間輪之「待研磨、研磨中、已完工備用」最新狀態與存放位置</p>
                </div>
            </div>
            
            <!-- 新增庫存面板 -->
            <div id="inventory-admin-panel" class="flex flex-wrap gap-2 items-center w-full xl:w-auto">
                <input type="text" id="inv-new-id" placeholder="庫存編號 (如 WR-009)" class="bg-slate-800 border border-slate-700 rounded-lg px-3 py-1.5 text-xs text-white placeholder-slate-500 focus:outline-none w-full md:w-36">
                <select id="inv-new-spec" class="bg-slate-800 border border-slate-700 rounded-lg px-3 py-1.5 text-xs text-white focus:outline-none w-full md:w-56">
                </select>
                <input type="text" id="inv-new-loc" placeholder="存放位置" class="bg-slate-800 border border-slate-700 rounded-lg px-3 py-1.5 text-xs text-white placeholder-slate-500 focus:outline-none w-full md:w-24" value="A架-01">
                <button onclick="addNewInventory()" class="bg-cyan-600 hover:bg-cyan-500 text-white px-3 py-1.5 rounded-lg font-bold transition-all text-xs flex items-center gap-1.5 cursor-pointer whitespace-nowrap shadow">
                    <i class="fa-solid fa-plus"></i> 新增庫存
                </button>
            </div>
        </div>
        
        <!-- 庫存篩選控制 -->
        <div class="p-4 bg-slate-50 border-b border-slate-200 flex flex-wrap justify-between items-center gap-3">
            <div class="bg-white p-1 rounded-lg border border-slate-200 flex gap-1 shadow-sm">
                <button onclick="filterInventory('全部')" id="btn-inv-全部" class="px-3 py-1 text-xs font-bold rounded bg-slate-700 text-white transition-all cursor-pointer">全部庫存</button>
                <button onclick="filterInventory('待研磨')" id="btn-inv-待研磨" class="px-3 py-1 text-xs font-bold rounded text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">待研磨毛胚</button>
                <button onclick="filterInventory('研磨中')" id="btn-inv-研磨中" class="px-3 py-1 text-xs font-bold rounded text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">研磨生產中</button>
                <button onclick="filterInventory('已完工')" id="btn-inv-已完工" class="px-3 py-1 text-xs font-bold rounded text-slate-500 hover:bg-slate-100 transition-all cursor-pointer">已完工備用</button>
            </div>
            <span class="text-xs text-slate-400 font-medium"><i class="fa-solid fa-circle-info text-cyan-500 mr-1"></i>快捷排程功能會自動在上方看板排入工單，並與該庫存物資綁定。</span>
        </div>

        <div class="overflow-x-auto">
            <table class="w-full text-left table-fixed min-w-[1000px]">
                <thead>
                    <tr class="bg-slate-100 text-slate-600 text-xs font-bold tracking-wider border-b border-slate-200">
                        <th class="p-3 w-32">庫存編號</th>
                        <th class="p-3 w-56">輪子細分規格</th>
                        <th class="p-3 w-36">目前庫存狀態</th>
                        <th class="p-3 w-32">存放位置</th>
                        <th class="p-3 w-40">最後研磨完成時間</th>
                        <th class="p-3 w-28 text-center action-col">庫存維護</th>
                        <th class="p-3 w-44 text-center action-col">快捷排程 (送至看板)</th>
                    </tr>
                </thead>
                <tbody id="inventory-table-body" class="divide-y divide-slate-100 text-sm text-slate-700">
                </tbody>
            </table>
        </div>
    </section>

    <!-- 歷史查詢 Modal -->
    <div id="history-modal" class="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-50 hidden flex items-center justify-center p-4">
        <div class="bg-white w-full max-w-5xl rounded-2xl border border-slate-200 shadow-2xl flex flex-col max-h-[85vh]">
            <div class="p-6 border-b border-slate-200 flex justify-between items-center bg-slate-50">
                <div class="flex items-center gap-3">
                    <i class="fa-solid fa-database text-amber-500 text-2xl"></i>
                    <div>
                        <h2 class="text-xl font-bold text-slate-800">歷史研磨資料與前幾日工單回查庫</h2>
                        <p class="text-xs text-slate-500 mt-0.5">回查過去已完成結案的舊工單</p>
                    </div>
                </div>
                <button onclick="toggleHistoryModal(false)" class="text-slate-400 hover:text-slate-600 text-2xl cursor-pointer p-2">&times;</button>
            </div>
            
            <div class="p-4 bg-slate-50 border-b border-slate-200 flex flex-wrap gap-2 items-center justify-between">
                <div class="flex gap-2">
                    <button onclick="filterHistoryByDays(0)" id="btn-day-0" class="px-3 py-1.5 text-xs font-bold rounded-lg bg-amber-600 text-white transition-all cursor-pointer">今天工單</button>
                    <button onclick="filterHistoryByDays(1)" id="btn-day-1" class="px-3 py-1.5 text-xs font-bold rounded-lg text-slate-600 bg-slate-200 hover:bg-slate-300 transition-all cursor-pointer">昨天工單</button>
                    <button onclick="filterHistoryByDays(2)" id="btn-day-2" class="px-3 py-1.5 text-xs font-bold rounded-lg text-slate-600 bg-slate-200 hover:bg-slate-300 transition-all cursor-pointer">前天工單</button>
                    <button onclick="filterHistoryByDays(7)" id="btn-day-7" class="px-3 py-1.5 text-xs font-bold rounded-lg text-slate-600 bg-slate-200 hover:bg-slate-300 transition-all cursor-pointer">過去一週</button>
                    <button onclick="filterHistoryByDays(-1)" id="btn-day-all" class="px-3 py-1.5 text-xs font-bold rounded-lg text-slate-600 bg-slate-200 hover:bg-slate-300 transition-all cursor-pointer">顯示全部歷史</button>
                </div>
                <div class="relative w-full md:w-64 mt-2 md:mt-0">
                    <i class="fa-solid fa-magnifying-glass absolute left-3 top-2.5 text-slate-400 text-xs"></i>
                    <input type="text" id="history-search" oninput="searchHistory()" class="w-full bg-white border border-slate-300 rounded-lg pl-8 pr-3 py-1.5 text-xs text-slate-700 placeholder-slate-400 focus:outline-none" placeholder="輸入關鍵字深層搜尋...">
                </div>
            </div>

            <div class="overflow-y-auto flex-1 p-4">
                <table class="w-full text-left border-collapse table-fixed min-w-[800px]">
                    <thead>
                        <tr class="text-slate-500 text-xs font-bold bg-slate-100 sticky top-0 border-b border-slate-200">
                            <th class="p-3 w-36">完工日期/時間</th>
                            <th class="p-3 w-16">機台</th>
                            <th class="p-3 w-48">工作輪細分規格</th>
                            <th class="p-3 w-16">番號</th>
                            <th class="p-3 w-20 text-center">責任班別</th>
                            <th class="p-3">工藝備註與歷史紀錄</th>
                            <th class="p-3 w-20 text-center">狀態</th>
                        </tr>
                    </thead>
                    <tbody id="history-table-body" class="divide-y divide-slate-100 text-sm text-slate-600">
                    </tbody>
                </table>
            </div>
            <div class="p-4 border-t border-slate-200 bg-slate-50 text-right text-xs text-slate-400 font-medium">
                目前條件下，共追查到 <span id="history-count" class="text-amber-600 font-bold">0</span> 筆研磨紀錄
            </div>
        </div>
    </div>

    <script>
        let currentRole = "leader"; 

        // 主排程看板資料[cite: 1]
        let kanbanData = [
            { id: 101, seq: 1, machine: "ZH1", type: "工作輪 WR (M2-20um)", process: "#60", shift: "A", note: "最小輪子 / 用於板面BA4", etf: "10:30", status: "進行中", urgent: false, targetDate: "today" },
            { id: 102, seq: 2, machine: "ZH1", type: "工作輪 WR (M2-10um)", process: "#120", shift: "A", note: "最小輪子", etf: "11:45", status: "待研磨", urgent: false, targetDate: "today" },
            { id: 103, seq: 3, machine: "ZH1", type: "工作輪 WR (粉末-20um)", process: "#120", shift: "B", note: "最小輪子 / 用於板面BA3", etf: "16:20", status: "待研磨", urgent: false, targetDate: "today" },
            { id: 105, seq: 4, machine: "ZH1", type: "中間輪 IMR 斜長140", process: "#60", shift: "A", note: "支撐BUR與工作輪中間", etf: "14:00", status: "品質檢驗", urgent: false, targetDate: "today" },
            { id: 401, seq: 1, machine: "ZH1", type: "工作輪 WR (粉末-平輪)", process: "#220", shift: "A", note: "【明日預排】早班備料", etf: "09:00", status: "待研磨", urgent: false, targetDate: "tomorrow" },
            { id: 402, seq: 2, machine: "TL", type: "中間輪 IMR / Φ60", process: "常態", shift: "A", note: "【明日預排】待磨數量多提早排入", etf: "11:00", status: "待研磨", urgent: false, targetDate: "tomorrow" }
        ];

        // 廠內全部庫存資料 (與上方主排程有 linkId 連動)
        let inventoryData = [
            { invId: "WR-001", type: "工作輪 WR (M2-20um)", status: "研磨中", location: "A架-01", lastGrind: "2026/07/16 09:30", linkId: 101 },
            { invId: "WR-002", type: "工作輪 WR (M2-10um)", status: "待研磨", location: "A架-02", lastGrind: "2026/07/15 14:15", linkId: 102 },
            { invId: "WR-003", type: "工作輪 WR (粉末-20um)", status: "待研磨", location: "B架-05", lastGrind: "2026/07/15 09:40", linkId: 103 },
            { invId: "IMR-140", type: "中間輪 IMR 斜長140", status: "待研磨", location: "C架-03", lastGrind: "2026/07/14 16:20", linkId: 105 },
            { invId: "WR-005", type: "工作輪 WR (M2-平輪)", status: "已完工", location: "成品備用區-A", lastGrind: "2026/07/16 08:15", linkId: null },
            { invId: "WR-006", type: "工作輪 WR (粉末-平輪)", status: "待研磨", location: "B架-12", lastGrind: "2026/07/14 11:30", linkId: 401 },
            { invId: "IMR-060", type: "中間輪 IMR / Φ60", status: "待研磨", location: "C架-08", lastGrind: "2026/07/13 15:50", linkId: 402 }
        ];

        function getPastDateStr(daysAgo) {
            const d = new Date(); d.setDate(d.getDate() - daysAgo);
            return `${d.getFullYear()}/${String(d.getMonth()+1).padStart(2,'0')}/${String(d.getDate()).padStart(2,'0')}`;
        }

        let historyData = [
            { id: 50, date: `${getPastDateStr(0)} 08:15`, machine: "ZH1", type: "工作輪 WR (M2-平輪)", process: "放電", shift: "A", note: "今日早班完工 / Ra符合標準", status: "已完成", daysAgo: 0 }
        ];

        let currentFilter = "全部"; 
        let currentInvFilter = "全部";
        let currentHistoryDayFilter = -1; 
        let viewDate = "today";

        const specMap = {
            ZH1: ["工作輪 WR (M2-20um)", "工作輪 WR (M2-10um)", "工作輪 WR (M2-平輪)", "工作輪 WR (粉末-20um)", "工作輪 WR (粉末-10um)", "工作輪 WR (粉末-平輪)", "中間輪 IMR 斜長140", "支撐輪 BUR"],
            TL: ["工作輪 WR / Φ20", "中間輪 IMR / Φ36", "中間輪 IMR / Φ60"],
            SP: ["工作輪 WR (1000/160)", "工作輪 WR (1000/50)", "工作輪 WR (60/50)", "工作輪 WR (320/50)"]
        };

        // 權限切換核心函數
        function switchRole(role) {
            currentRole = role;
            const badge = document.getElementById("role-badge");
            const adminPanel = document.getElementById("admin-dispatch-panel");
            const hintMessage = document.getElementById("role-hint-message");
            const moveBtn = document.getElementById("btn-move-tomorrow-to-today");

            if (role === "leader") {
                badge.className = "px-2.5 py-0.5 rounded-full text-xs font-bold bg-emerald-100 text-emerald-700 border border-emerald-200";
                badge.innerHTML = `<i class="fa-solid fa-user-shield mr-1"></i>作業長權限`;
                adminPanel.classList.remove("hidden");
                moveBtn.classList.remove("hidden"); 
                hintMessage.innerText = "提示：時間已全面換裝為手動鍵入模式，可自由輸入 24H 制數值（如 06:15、23:40）";
            } else {
                badge.className = "px-2.5 py-0.5 rounded-full text-xs font-bold bg-slate-200 text-slate-600 border border-slate-300";
                badge.innerHTML = `<i class="fa-solid fa-user mr-1"></i>一般作業員 (唯讀)`;
                adminPanel.classList.add("hidden"); 
                moveBtn.classList.add("hidden"); 
                hintMessage.innerText = "🔒 您目前為唯讀狀態，僅能查看工單動態。如需排單請聯絡作業長。";
                
                if (viewDate !== "today") {
                    switchDateView("today");
                    return; 
                }
            }

            // 重新渲染兩個表
            renderKanbanTable();
            renderInventoryTable();

            // 批次開關作業行動作欄
            const actionCols = document.querySelectorAll(".action-col");
            if (role === "leader") {
                actionCols.forEach(el => el.classList.remove("hidden"));
            } else {
                actionCols.forEach(el => el.classList.add("hidden"));
            }
        }

        // 一鍵移轉明日預排
        function moveTomorrowToToday() {
            if (currentRole !== "leader") return alert("權限不足！只有作業長可以執行此操作。");
            let tomorrowOrders = kanbanData.filter(item => item.targetDate === "tomorrow");
            if (tomorrowOrders.length === 0) return alert("目前「明日預排工單」中沒有任何資料可供移轉。");

            if (confirm(`確定要將目前的 ${tomorrowOrders.length} 筆明日預排工單全部接續到「今日動態排程」的末尾嗎？`)) {
                let todayOrders = kanbanData.filter(item => item.targetDate === "today");
                let maxSeq = todayOrders.length > 0 ? Math.max(...todayOrders.map(o => o.seq)) : 0;

                tomorrowOrders.sort((a, b) => a.seq - b.seq);
                tomorrowOrders.forEach(item => {
                    maxSeq++;
                    item.targetDate = "today";
                    item.seq = maxSeq;
                    item.note = item.note.replace("【明日預排】", "📋 ");
                });

                switchDateView("today");
                alert("移轉成功！明日預排工單已全數拼入今日動態排程。");
            }
        }

        function adaptSpecOptions() {
            const mac = document.getElementById("ins-machine").value;
            const specSelect = document.getElementById("ins-spec");
            specSelect.innerHTML = "";
            specMap[mac].forEach(s => {
                const opt = document.createElement("option"); opt.value = s; opt.innerText = s;
                specSelect.appendChild(opt);
            });
        }

        // 彙總全規格至庫存新增選單
        function populateInventorySpecs() {
            const select = document.getElementById("inv-new-spec");
            select.innerHTML = "";
            Object.keys(specMap).forEach(mac => {
                specMap[mac].forEach(spec => {
                    const opt = document.createElement("option");
                    opt.value = spec;
                    opt.innerText = `[${mac}] ${spec}`;
                    select.appendChild(opt);
                });
            });
        }

        function switchDateView(dateType) {
            if (currentRole === "operator" && dateType === "tomorrow") {
                alert("一般作業員權限僅能查看「今日動態排程」，「明日預排」只有作業長能查看與調整。");
                return;
            }

            viewDate = dateType;
            const btnToday = document.getElementById("tab-date-today");
            const btnTomorrow = document.getElementById("tab-date-tomorrow");
            const title = document.getElementById("current-date-view-title");
            const labels = document.querySelectorAll(".target-date-label");

            if (dateType === "today") {
                btnToday.className = "px-5 py-2 text-sm font-bold rounded-lg bg-white text-slate-800 shadow transition-all cursor-pointer flex items-center gap-2";
                btnTomorrow.className = "px-5 py-2 text-sm font-bold rounded-lg text-slate-600 hover:bg-slate-100/60 transition-all cursor-pointer flex items-center gap-2";
                title.innerText = "今日動態排程"; title.className = "text-emerald-600 font-extrabold text-base";
                labels.forEach(l => l.innerText = "今日");
            } else {
                btnToday.className = "px-5 py-2 text-sm font-bold rounded-lg text-slate-600 hover:bg-slate-100/60 transition-all cursor-pointer flex items-center gap-2";
                btnTomorrow.className = "px-5 py-2 text-sm font-bold rounded-lg bg-white text-slate-800 shadow transition-all cursor-pointer flex items-center gap-2";
                title.innerText = "明日預排工單"; title.className = "text-amber-600 font-extrabold text-base";
                labels.forEach(l => l.innerText = "明日");
            }
            document.getElementById("ins-date").value = dateType;
            
            renderKanbanTable();
            // 切換時再次確保權限對應欄位顯隱一致
            const actionCols = document.querySelectorAll(".action-col");
            if (currentRole === "leader") {
                actionCols.forEach(el => el.classList.remove("hidden"));
            } else {
                actionCols.forEach(el => el.classList.add("hidden"));
            }
        }

        function filterStatus(statusType) {
            currentFilter = statusType;
            document.getElementById("current-filter-label").innerText = statusType;
            ["全部", "待研磨", "進行中", "已完工"].forEach(t => {
                const btn = document.getElementById(`btn-filter-${t}`);
                btn.className = t === statusType ? "px-4 py-1.5 text-sm font-bold rounded-lg bg-cyan-600 text-white transition-all cursor-pointer shadow-sm" : "px-4 py-1.5 text-sm font-bold rounded-lg text-slate-500 hover:bg-slate-100 transition-all cursor-pointer";
            });
            renderKanbanTable();
        }

        function updateDashboardCounters(activeViewData) {
            let total = activeViewData.length; let waiting = 0; let processing = 0; let completed = 0;
            let activeGrindingNames = [];

            activeViewData.forEach(item => {
                if (item.status === "待研磨") waiting++;
                else if (item.status === "進行中" || item.status === "品質檢驗") {
                    processing++;
                    activeGrindingNames.push(`${item.machine}-${item.type.replace("工作輪 WR ", "").replace("中間輪 IMR ", "")}`);
                }
                else if (item.status === "已完工") completed++;
            });

            document.getElementById("dash-total").innerText = total + " 組";
            document.getElementById("dash-waiting").innerText = waiting + " 組";
            document.getElementById("dash-processing").innerText = processing + " 組";
            document.getElementById("dash-completed").innerText = completed + " 組";

            const nameLabel = document.getElementById("dash-processing-names");
            if (activeGrindingNames.length > 0) {
                nameLabel.innerText = "⚡ " + activeGrindingNames.join(", ");
                nameLabel.className = "text-xs text-cyan-600 mt-1 font-bold truncate max-w-[220px] bg-cyan-50 p-1 rounded border border-cyan-200/50 inline-block";
            } else {
                nameLabel.innerText = "暫無研磨任務"; nameLabel.className = "text-xs text-slate-400 mt-1 font-medium truncate max-w-full";
            }
        }

        // 渲染主排程看板[cite: 1]
        function renderKanbanTable() {
            let activeViewData = kanbanData.filter(x => x.targetDate === viewDate);
            activeViewData.sort((a, b) => a.seq - b.seq);
            const tbody = document.getElementById("kanban-table-body");
            tbody.innerHTML = "";

            updateDashboardCounters(activeViewData);

            let filtered = activeViewData.filter(item => {
                if (currentFilter === "全部") return true;
                if (currentFilter === "待研磨") return item.status === "待研磨";
                if (currentFilter === "進行中") return item.status === "進行中" || item.status === "品質檢驗";
                if (currentFilter === "已完工") return item.status === "已完工";
            });

            const isLeader = (currentRole === "leader");

            filtered.forEach((item, index) => {
                let macBadge = "bg-slate-100 text-slate-600";
                if (item.machine === "ZH1") macBadge = "bg-emerald-50 text-emerald-600 border border-emerald-200";
                else if (item.machine === "TL") macBadge = "bg-blue-50 text-blue-600 border border-blue-200";
                else if (item.machine === "SP") macBadge = "bg-purple-50 text-purple-600 border border-purple-200";

                let rowBg = 'hover:bg-slate-50';
                if (item.urgent) {
                    rowBg = 'bg-red-50 hover:bg-red-100/60 font-medium';
                } else if (item.status === '已完工') {
                    rowBg = 'bg-emerald-50/40 hover:bg-emerald-100/50';
                } else if (item.status === '進行中') {
                    rowBg = 'bg-cyan-50/20 hover:bg-cyan-100/30';
                }

                const tr = document.createElement("tr");
                tr.className = `transition-all text-slate-700 ${rowBg}`;

                let rowHtml = `
                    <td class="p-4 text-center font-bold text-base ${item.urgent?'text-red-600':'text-slate-600'}">${item.seq}</td>
                    <td class="p-4 text-center"><span class="px-2 py-0.5 rounded-md text-xs font-bold ${macBadge}">${item.machine}</span></td>
                    <td class="p-4 font-bold text-slate-800 truncate max-w-[200px]" title="${item.type}">${item.type}</td>
                    <td class="p-4"><span class="bg-slate-100 text-cyan-700 px-2 py-0.5 rounded font-mono text-xs font-bold border border-slate-200">${item.process}</span></td>
                `;

                if (isLeader) {
                    rowHtml += `
                        <td class="p-4">
                            <select onchange="updateItemField(${item.id}, 'shift', this.value)" class="bg-white border border-slate-300 rounded px-1.5 py-0.5 text-xs text-slate-700 focus:outline-none">
                                <option value="A" ${item.shift==='A'?'selected':''}>A班(早)</option>
                                <option value="B" ${item.shift==='B'?'selected':''}>B班(中)</option>
                                <option value="C" ${item.shift==='C'?'selected':''}>C班(夜)</option>
                            </select>
                        </td>
                        <td class="p-4 text-xs text-slate-500 truncate max-w-[220px]" title="${item.note}">${item.note || '-'}</td>
                        <td class="p-4">
                            <div class="flex items-center gap-1">
                                <span class="text-slate-400 text-xs"><i class="fa-regular fa-clock"></i></span>
                                <input type="text" value="${item.etf}" onchange="updateItemField(${item.id}, 'etf', this.value)" 
                                    class="bg-white border border-slate-300 rounded-lg p-1 text-emerald-600 font-mono font-extrabold text-xl text-center w-28 focus:outline-none focus:border-emerald-500 shadow-sm">
                            </div>
                        </td>
                        <td class="p-4">
                            <div class="flex items-center gap-1.5">
                                <select onchange="updateStatus(${item.id}, this.value)" class="bg-white border border-slate-3 p-1 text-xs font-bold ${item.status==='已完工'?'text-emerald-600 border-emerald-300':'text-slate-700'} focus:outline-none">
                                    <option value="待研磨" ${item.status==='待研磨'?'selected':''}>待研磨</option>
                                    <option value="進行中" ${item.status==='進行中'?'selected':''}>進行中</option>
                                    <option value="品質檢驗" ${item.status==='品質檢驗'?'selected':''}>品質檢驗</option>
                                    <option value="已完工" ${item.status==='已完工'?'selected':''}>已完工</option>
                                </select>
                                ${item.status === '進行中' ? `
                                    <button onclick="quickCompleteGrinding(${item.id})" class="bg-emerald-600 hover:bg-emerald-500 text-white px-1.5 py-1 rounded text-[10px] font-extrabold cursor-pointer transition-all shadow animate-pulse whitespace-nowrap" title="一鍵完成研磨">
                                        <i class="fa-solid fa-circle-check"></i> 完成
                                    </button>
                                ` : ''}
                            </div>
                        </td>
                        <td class="p-4 text-center">
                            <div class="inline-flex rounded-md shadow-sm" role="group">
                                <button onclick="moveOrder(${item.id}, -1)" class="px-2 py-1 text-xs font-medium text-slate-700 bg-white border border-slate-300 rounded-l-lg hover:bg-slate-100 cursor-pointer">
                                    <i class="fa-solid fa-arrow-up text-cyan-600"></i>
                                </button>
                                <button onclick="moveOrder(${item.id}, 1)" class="px-2 py-1 text-xs font-medium text-slate-700 bg-white border-l-0 border border-slate-300 rounded-r-lg hover:bg-slate-100 cursor-pointer">
                                    <i class="fa-solid fa-arrow-down text-slate-500"></i>
                                </button>
                            </div>
                        </td>
                        <td class="p-4 text-center">
                            <button onclick="deleteItem(${item.id})" class="bg-red-50 hover:bg-red-100 text-red-600 border border-red-200 px-2 py-1 rounded text-xs font-bold cursor-pointer transition-colors">
                                <i class="fa-solid fa-trash-can mr-1"></i>刪除
                            </button>
                        </td>
                        <td class="p-4 text-center">
                            <button onclick="archiveItem(${item.id})" class="bg-emerald-50 hover:bg-emerald-100 text-emerald-600 border border-emerald-200 px-2 py-1 rounded text-xs font-bold cursor-pointer transition-colors">
                                <i class="fa-solid fa-box-archive mr-1"></i>結案
                            </button>
                        </td>
                    `;
                } else {
                    let statusColor = "text-slate-600 bg-slate-100";
                    if (item.status === "進行中") statusColor = "text-cyan-700 bg-cyan-50 border border-cyan-200";
                    else if (item.status === "品質檢驗") statusColor = "text-purple-700 bg-purple-50 border border-purple-200";
                    else if (item.status === "已完工") statusColor = "text-emerald-700 bg-emerald-50 border border-emerald-200";

                    rowHtml += `
                        <td class="p-4"><span class="px-2 py-1 text-xs bg-slate-100 rounded text-slate-700 font-bold">${item.shift}班</span></td>
                        <td class="p-4 text-xs text-slate-500 truncate max-w-[220px]" title="${item.note}">${item.note || '-'}</td>
                        <td class="p-4 font-mono font-extrabold text-xl text-emerald-600">${item.etf}</td>
                        <td class="p-4"><span class="px-2 py-1 rounded text-xs font-bold ${statusColor}">${item.status}</span></td>
                        <td class="hidden"></td><td class="hidden"></td><td class="hidden"></td> `;
                }

                tr.innerHTML = rowHtml;
                tbody.appendChild(tr);
            });
        }

        function moveOrder(id, direction) {
            if (currentRole !== "leader") return alert("權限不足！只有作業長可以調整排程順序。");
            let activeViewData = kanbanData.filter(x => x.targetDate === viewDate); activeViewData.sort((a, b) => a.seq - b.seq);
            const idx = activeViewData.findIndex(x => x.id === id); if (idx === -1) return;
            const targetIdx = idx + direction; if (targetIdx < 0 || targetIdx >= activeViewData.length) return;

            let tempSeq = activeViewData[idx].seq;
            activeViewData[idx].seq = activeViewData[targetIdx].seq; activeViewData[targetIdx].seq = tempSeq;
            renderKanbanTable();
        }

        function deleteItem(id) {
            if (currentRole !== "leader") return alert("權限不足！只有作業長可以刪除工單。");
            const item = kanbanData.find(x => x.id === id); if (!item) return;
            if (confirm(`確認要直接「刪除」這筆工單嗎？\n👉 規格：${item.type}`)) {
                // 如果刪除，將連動的庫存狀態釋放回待研磨
                let invItem = inventoryData.find(inv => inv.linkId === id);
                if (invItem) {
                    invItem.linkId = null;
                    invItem.status = "待研磨";
                }

                kanbanData = kanbanData.filter(x => x.id !== id);
                let activeViewData = kanbanData.filter(x => x.targetDate === item.targetDate);
                activeViewData.sort((a, b) => a.seq - b.seq); activeViewData.forEach((x, i) => { x.seq = i + 1; });
                
                renderKanbanTable();
                renderInventoryTable();
                alert("工單已成功刪除！");
            }
        }

        // 核心邏輯：新增工單與緊急插單[cite: 1]
        function insertWorkOrder(isUrgent) {
            if (currentRole !== "leader") return alert("權限不足！只有作業長可以指派或新增工單。");
            const targetDate = document.getElementById("ins-date").value;
            let seq = parseInt(document.getElementById("ins-seq").value) || 1;
            const machine = document.getElementById("ins-machine").value;
            const type = document.getElementById("ins-spec").value;
            const process = document.getElementById("ins-process").value || "#60";
            const etf = document.getElementById("ins-etf").value; 

            if (!type) return alert("規格錯誤");
            let datePrefix = targetDate === "tomorrow" ? "【明日預排】" : "📋 ";
            let note = isUrgent ? "⚠️ 緊急插單" : datePrefix + "排程派單";
            if (type.includes("M2-20um")) note += " / BA4板面";
            if (type.includes("粉末-平輪")) note += " / 最後1PASS";

            // 先取得該日的當前所有排單並進行排序
            let activeViewData = kanbanData.filter(x => x.targetDate === targetDate);
            activeViewData.sort((a, b) => a.seq - b.seq);

            if (isUrgent) {
                // 【核心修正點】：尋找目前正在「進行中」的工單索引
                const runningIdx = activeViewData.findIndex(x => x.status === "進行中");
                
                if (runningIdx !== -1) {
                    // 若找到進行中，緊急插單排在「進行中」的下一順位 (running + 1)
                    seq = activeViewData[runningIdx].seq + 1;
                    // 將後續所有工單順序往後推1
                    kanbanData.forEach(item => {
                        if (item.targetDate === targetDate && item.seq >= seq) {
                            item.seq += 1;
                        }
                    });
                } else {
                    // 若當下無進行中的任務，直接置於第1順位
                    seq = 1;
                    kanbanData.forEach(item => {
                        if (item.targetDate === targetDate && item.seq >= 1) {
                            item.seq += 1;
                        }
                    });
                }
            } else {
                // 一般派單：指定位置以後的所有工單往後退1
                kanbanData.forEach(item => {
                    if (item.targetDate === targetDate && item.seq >= seq) {
                        item.seq += 1;
                    }
                });
            }

            kanbanData.push({
                id: Date.now(), seq: seq, machine: machine, type: type, process: process,
                shift: "A", note: note, etf: etf, status: "待研磨", urgent: isUrgent, targetDate: targetDate
            });
            
            switchDateView(targetDate);
        }

        // 一鍵完成研磨快捷鍵 (強化待研磨極完工功能)
        function quickCompleteGrinding(id) {
            if (currentRole !== "leader") return alert("權限不足！");
            updateStatus(id, "已完工");
            alert("🎉 研磨完成！已同步更新庫存狀態為【已完工備用】。");
        }

        function updateItemField(id, field, value) { 
            if (currentRole !== "leader") return;
            const item = kanbanData.find(x => x.id === id); if (item) item[field] = value; 
        }
        
        function updateStatus(id, value) { 
            if (currentRole !== "leader") return;
            const item = kanbanData.find(x => x.id === id); 
            if (item) { 
                item.status = value; 
                
                // 🔄 同步連動庫存狀態
                let invItem = inventoryData.find(inv => inv.linkId === id);
                if (invItem) {
                    if (value === "進行中") {
                        invItem.status = "研磨中";
                    } else if (value === "已完工") {
                        invItem.status = "已完工";
                        const now = new Date();
                        invItem.lastGrind = `${now.getFullYear()}/${String(now.getMonth()+1).padStart(2,'0')}/${String(now.getDate()).padStart(2,'0')} ${String(now.getHours()).padStart(2,'0')}:${String(now.getMinutes()).padStart(2,'0')}`;
                    } else if (value === "待研磨" || value === "品質檢驗") {
                        invItem.status = "待研磨";
                    }
                }
                
                renderKanbanTable(); 
                renderInventoryTable();
            } 
        }

        function archiveItem(id) {
            if (currentRole !== "leader") return alert("權限不足！只有作業長可以執行結案封存。");
            const itemIndex = kanbanData.findIndex(x => x.id === id); if (itemIndex === -1) return;
            const item = kanbanData[itemIndex]; if (item.status !== "已完工" && !confirm("尚未完工，確認強制封存嗎？")) return;

            const now = new Date();
            const timeStr = `${now.getFullYear()}/${String(now.getMonth()+1).padStart(2,'0')}/${String(now.getDate()).padStart(2,'0')} ${String(now.getHours()).padStart(2,'0')}:${String(now.getMinutes()).padStart(2,'0')}`;

            historyData.unshift({
                id: Date.now(), date: timeStr, machine: item.machine, type: item.type, process: item.process,
                shift: item.shift, note: item.note + " (封存)", status: "已完成", daysAgo: 0
            });

            // 🔄 結案解綁庫存，將庫存品設為完工儲備
            let invItem = inventoryData.find(inv => inv.linkId === id);
            if (invItem) {
                invItem.status = "已完工";
                invItem.lastGrind = timeStr;
                invItem.linkId = null; // 解除實時繫結
            }

            kanbanData.splice(itemIndex, 1);
            let activeViewData = kanbanData.filter(x => x.targetDate === item.targetDate);
            activeViewData.sort((a, b) => a.seq - b.seq); activeViewData.forEach((x, i) => { x.seq = i + 1; });

            renderKanbanTable(); 
            renderHistoryTable();
            renderInventoryTable();
            alert("工單已成功封存！庫存轉為【已完工備用】狀態。");
        }

        /* ================= 📦 全部庫存表邏輯 (新增部分) ================= */
        
        function filterInventory(status) {
            currentInvFilter = status;
            ["全部", "待研磨", "研磨中", "已完工"].forEach(t => {
                const btn = document.getElementById(`btn-inv-${t}`);
                btn.className = t === status ? 
                    "px-3 py-1 text-xs font-bold rounded bg-slate-700 text-white transition-all cursor-pointer shadow-sm" : 
                    "px-3 py-1 text-xs font-bold rounded text-slate-500 hover:bg-slate-100 transition-all cursor-pointer";
            });
            renderInventoryTable();
        }

        function renderInventoryTable() {
            const tbody = document.getElementById("inventory-table-body");
            tbody.innerHTML = "";

            let filtered = inventoryData;
            if (currentInvFilter !== "全部") {
                filtered = inventoryData.filter(inv => inv.status === currentInvFilter);
            }

            const isLeader = (currentRole === "leader");
            const invAdminPanel = document.getElementById("inventory-admin-panel");
            
            if (isLeader) {
                invAdminPanel.classList.remove("hidden");
            } else {
                invAdminPanel.classList.add("hidden");
            }

            filtered.forEach(inv => {
                let statusBadge = "";
                if (inv.status === "待研磨") {
                    statusBadge = "<span class='px-2 py-0.5 rounded text-xs font-bold bg-slate-100 text-slate-600 border border-slate-300'>待研磨毛胚</span>";
                } else if (inv.status === "研磨中") {
                    statusBadge = "<span class='px-2 py-0.5 rounded text-xs font-bold bg-cyan-100 text-cyan-700 border border-cyan-300 urgent-pulse inline-block'>研磨生產中</span>";
                } else if (inv.status === "已完工") {
                    statusBadge = "<span class='px-2 py-0.5 rounded text-xs font-bold bg-emerald-100 text-emerald-700 border border-emerald-300'>已完工備用</span>";
                }

                const tr = document.createElement("tr");
                tr.className = "hover:bg-slate-50 transition-colors";

                let rowHtml = `
                    <td class="p-3 font-mono font-bold text-slate-700">${inv.invId}</td>
                    <td class="p-3 font-semibold text-slate-800 truncate max-w-[220px]" title="${inv.type}">${inv.type}</td>
                    <td class="p-3">${statusBadge}</td>
                `;

                if (isLeader) {
                    rowHtml += `
                        <td class="p-3">
                            <input type="text" value="${inv.location}" onchange="updateInvField('${inv.invId}', 'location', this.value)" class="bg-white border border-slate-300 rounded p-1 text-xs text-slate-700 focus:outline-none w-24">
                        </td>
                        <td class="p-3 font-mono text-xs text-slate-400">${inv.lastGrind}</td>
                        <td class="p-3 text-center action-col">
                            <button onclick="deleteInventory('${inv.invId}')" class="bg-red-50 hover:bg-red-100 text-red-600 border border-red-200 px-2 py-1 rounded text-xs font-bold cursor-pointer transition-colors">
                                <i class="fa-solid fa-trash-can mr-1"></i>報廢
                            </button>
                        </td>
                        <td class="p-3 text-center action-col">
                            <div class="inline-flex gap-1.5">
                                <button onclick="quickDispatchInventory('${inv.invId}', false)" class="bg-emerald-50 hover:bg-emerald-100 text-emerald-700 border border-emerald-200 px-2.5 py-1 rounded text-xs font-bold cursor-pointer transition-all flex items-center gap-1" title="排入末尾">
                                    <i class="fa-solid fa-calendar-plus"></i> 一般排程
                                </button>
                                <button onclick="quickDispatchInventory('${inv.invId}', true)" class="bg-red-50 hover:bg-red-100 text-red-600 border border-red-200 px-2.5 py-1 rounded text-xs font-bold cursor-pointer transition-all flex items-center gap-1" title="排在研磨中下一順位">
                                    <i class="fa-solid fa-bolt"></i> 緊急插單
                                </button>
                            </div>
                        </td>
                    `;
                } else {
                    rowHtml += `
                        <td class="p-3 text-xs text-slate-500">${inv.location}</td>
                        <td class="p-3 font-mono text-xs text-slate-400">${inv.lastGrind}</td>
                        <td class="hidden"></td><td class="hidden"></td>
                    `;
                }

                tr.innerHTML = rowHtml;
                tbody.appendChild(tr);
            });
        }

        function updateInvField(invId, field, value) {
            const inv = inventoryData.find(x => x.invId === invId);
            if (inv) inv[field] = value;
        }

        function addNewInventory() {
            if (currentRole !== "leader") return alert("權限不足！");
            const invId = document.getElementById("inv-new-id").value.trim().toUpperCase();
            const type = document.getElementById("inv-new-spec").value;
            const loc = document.getElementById("inv-new-loc").value.trim() || "未定位";

            if (!invId) return alert("請輸入庫存編號！");
            if (inventoryData.some(inv => inv.invId === invId)) return alert("庫存編號重複！");

            inventoryData.unshift({
                invId: invId,
                type: type,
                status: "待研磨",
                location: loc,
                lastGrind: "無紀錄",
                linkId: null
            });

            document.getElementById("inv-new-id").value = "";
            renderInventoryTable();
            alert(`成功新增庫存件: ${invId}`);
        }

        function deleteInventory(invId) {
            if (currentRole !== "leader") return alert("權限不足！");
            if (confirm(`確認要報廢/刪除庫存編號 【${invId}】 嗎？`)) {
                inventoryData = inventoryData.filter(inv => inv.invId !== invId);
                renderInventoryTable();
            }
        }

        // 從全部庫存表快捷建立排程
        function quickDispatchInventory(invId, isUrgent = false) {
            if (currentRole !== "leader") return alert("權限不足！");
            const invItem = inventoryData.find(inv => inv.invId === invId);
            if (!invItem) return;

            if (invItem.status === "研磨中") {
                alert("該項目正在研磨生產中，無法重複排程。");
                return;
            }

            // 智能判斷機台
            let recommendedMachine = "ZH1";
            if (invItem.type.includes("Φ20") || invItem.type.includes("Φ36") || invItem.type.includes("Φ60")) {
                recommendedMachine = "TL";
            } else if (invItem.type.includes("1000/") || invItem.type.includes("320/")) {
                recommendedMachine = "SP";
            }

            const actionText = isUrgent ? "🔥 【緊急插單】(插在目前研磨中下方)" : "📋 【一般排程】(排在今日末尾)";
            const confirmDispatch = confirm(`確認要將庫存【${invItem.invId}】進行 ${actionText} 嗎？`);
            
            if (confirmDispatch) {
                const newId = Date.now();
                let todayOrders = kanbanData.filter(x => x.targetDate === "today");
                todayOrders.sort((a, b) => a.seq - b.seq);

                let seq = todayOrders.length + 1;
                if (isUrgent) {
                    const runningIdx = todayOrders.findIndex(x => x.status === "進行中");
                    if (runningIdx !== -1) {
                        seq = todayOrders[runningIdx].seq + 1;
                        kanbanData.forEach(item => {
                            if (item.targetDate === "today" && item.seq >= seq) {
                                item.seq += 1;
                            }
                        });
                    } else {
                        seq = 1;
                        kanbanData.forEach(item => {
                            if (item.targetDate === "today" && item.seq >= 1) {
                                item.seq += 1;
                            }
                        });
                    }
                }

                kanbanData.push({
                    id: newId, 
                    seq: seq, 
                    machine: recommendedMachine, 
                    type: invItem.type, 
                    process: "#60",
                    shift: "A", 
                    note: isUrgent ? `⚠️ 庫存緊急插單 (${invItem.invId})` : `📦 庫存轉排單 (${invItem.invId})`, 
                    etf: "12:00", 
                    status: "待研磨", 
                    urgent: isUrgent, 
                    targetDate: "today"
                });

                // 綁定連動
                invItem.linkId = newId;
                invItem.status = "待研磨";

                switchDateView("today");
                renderInventoryTable();
                alert(`成功！庫存已順利關聯排程。`);
            }
        }

        /* ================= 歷史與視窗控制 ================= */

        function toggleHistoryModal(show) {
            const modal = document.getElementById("history-modal");
            if (show) { modal.classList.remove("hidden"); filterHistoryByDays(0); } else { modal.classList.add("hidden"); }
        }

        function filterHistoryByDays(days) {
            currentHistoryDayFilter = days;
            [0, 1, 2, 7, -1].forEach(d => {
                const idStr = d === -1 ? 'all' : d; const btn = document.getElementById(`btn-day-${idStr}`);
                btn.className = d === days ? "px-3 py-1.5 text-xs font-bold rounded-lg bg-amber-600 text-white transition-all cursor-pointer shadow-sm" : "px-3 py-1.5 text-xs font-bold rounded-lg text-slate-600 bg-slate-200 hover:bg-slate-300 transition-all cursor-pointer";
            });
            searchHistory();
        }

        function searchHistory() {
            const query = document.getElementById("history-search").value.toLowerCase();
            let filtered = historyData.filter(item => {
                if (currentHistoryDayFilter === 0) return item.daysAgo === 0;
                if (currentHistoryDayFilter === 1) return item.daysAgo === 1;
                if (currentHistoryDayFilter === 2) return item.daysAgo === 2;
                if (currentHistoryDayFilter === 7) return item.daysAgo <= 7; return true;
            });
            if (query) {
                filtered = filtered.filter(item => 
                    item.machine.toLowerCase().includes(query) || item.type.toLowerCase().includes(query) ||
                    item.process.toLowerCase().includes(query) || item.shift.toLowerCase().includes(query) ||
                    item.note.toLowerCase().includes(query)
                );
            }
            renderHistoryTable(filtered);
        }

        function renderHistoryTable(dataToRender) {
            const tbody = document.getElementById("history-table-body"); tbody.innerHTML = "";
            document.getElementById("history-count").innerText = dataToRender.length;
            if(dataToRender.length === 0) {
                tbody.innerHTML = `<tr><td colspan="7" class="p-8 text-center text-slate-400 font-medium">此篩選條件下，查無歷史工單紀錄。</td></tr>`; return;
            }
            dataToRender.forEach(item => {
                const tr = document.createElement("tr"); tr.className = "hover:bg-slate-50 text-slate-700";
                tr.innerHTML = `
                    <td class="p-3 font-mono text-xs text-slate-400">${item.date}</td>
                    <td class="p-3 font-bold text-cyan-600">${item.machine}</td>
                    <td class="p-3 font-semibold text-slate-800 truncate max-w-[180px]" title="${item.type}">${item.type}</td>
                    <td class="p-3"><span class="bg-slate-100 text-slate-600 px-1.5 py-0.5 rounded font-mono text-xs border border-slate-200">${item.process}</span></td>
                    <td class="p-3 text-center"><span class="bg-slate-100 px-2 py-0.5 rounded text-xs font-medium">${item.shift}班</span></td>
                    <td class="p-3 text-xs text-slate-500 truncate max-w-[250px]" title="${item.note}">${item.note}</td>
                    <td class="p-3 text-center"><span class="text-xs bg-emerald-50 text-emerald-600 border border-emerald-200 px-2 py-0.5 rounded-full font-bold">✓ 完工</span></td>
                `;
                tbody.appendChild(tr);
            });
        }

        function startClock() {
            setInterval(() => {
                const now = new Date();
                const year = now.getFullYear();
                const month = String(now.getMonth() + 1).padStart(2, '0');
                const date = String(now.getDate()).padStart(2, '0');
                const hours = String(now.getHours()).padStart(2, '0');
                const minutes = String(now.getMinutes()).padStart(2, '0');
                const seconds = String(now.getSeconds()).padStart(2, '0');
                
                document.getElementById('current-time').innerText = `${year}/${month}/${date} ${hours}:${minutes}:${seconds}`;
                
                const h = now.getHours(); 
                let shift = "C班 (夜班)";
                if (h >= 8 && h < 16) shift = "A班 (早班)"; 
                else if (h >= 16 && h < 24) shift = "B班 (中班)";
                document.getElementById('current-shift').innerText = shift;
            }, 1000);
        }

        window.onload = () => { 
            startClock(); 
            adaptSpecOptions(); 
            populateInventorySpecs();
            switchRole("leader"); 
        };
    </script>
</body>
</html>
