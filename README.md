<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cloud Note & Finance App</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <!-- Firebase SDKs (Compat version) -->
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap');
        body { font-family: 'Inter', sans-serif; overflow: hidden; }
        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 10px; }
        .line-clamp-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
    </style>
</head>
<body class="bg-slate-50 text-slate-900">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        // --- CẤU HÌNH FIREBASE CỦA BẠN ---
        const firebaseConfig = {
            apiKey: "AIzaSyBWW23jt2_V9o-z6NRCGw7Tqkjd3k6AFv4",
            authDomain: "mynote2026-9efd7.firebaseapp.com",
            projectId: "mynote2026-9efd7",
            storageBucket: "mynote2026-9efd7.firebasestorage.app",
            messagingSenderId: "855959444266",
            appId: "1:855959444266:web:12241f16c90387453c1c53",
            measurementId: "G-NN5ZEVQ01Z"
        };

        const TABS = {
            finance: { id: 'finance', label: 'Thu Chi', icon: 'wallet', color: 'text-amber-600', bg: 'bg-amber-100' },
            todos: { id: 'todos', label: 'Việc Cần Làm', icon: 'check-square', color: 'text-orange-600', bg: 'bg-orange-100' },
            subscribers: { id: 'subscribers', label: 'Nhóm Khách Hàng', icon: 'users', color: 'text-pink-600', bg: 'bg-pink-100' },
            courses: { id: 'courses', label: 'Khóa Học', icon: 'book-open', color: 'text-blue-600', bg: 'bg-blue-100' },
            consulting: { id: 'consulting', label: 'Tư Vấn & Note', icon: 'message-circle', color: 'text-purple-600', bg: 'bg-purple-100' },
            links: { id: 'links', label: 'Kho Link', icon: 'link', color: 'text-emerald-600', bg: 'bg-emerald-100' },
        };

        const Icon = ({ name, size = 20, className = "" }) => {
            useEffect(() => { lucide.createIcons(); }, [name]);
            return <i data-lucide={name} style={{ width: size, height: size }} className={className}></i>;
        };

        function App() {
            const [items, setItems] = useState([]);
            const [loading, setLoading] = useState(true);
            const [activeTab, setActiveTab] = useState('finance');
            const [searchTerm, setSearchTerm] = useState('');
            const [isMobileMenuOpen, setIsMobileMenuOpen] = useState(false);
            const [editingId, setEditingId] = useState(null);
            const [isBulkMode, setIsBulkMode] = useState(false);
            const [expandedItems, setExpandedItems] = useState({});
            const [copiedId, setCopiedId] = useState(null);

            const [title, setTitle] = useState('');
            const [content, setContent] = useState('');
            const [amount, setAmount] = useState('');
            const [transactionType, setTransactionType] = useState('expense');
            const [date, setDate] = useState(new Date().toISOString().split('T')[0]);
            const [endDate, setEndDate] = useState('');

            const dbRef = useRef(null);
            const userRef = useRef(null);

            useEffect(() => {
                if (!firebase.apps.length) firebase.initializeApp(firebaseConfig);
                
                firebase.auth().signInAnonymously().then(({ user }) => {
                    userRef.current = user;
                    const db = firebase.firestore();
                    dbRef.current = db;

                    const appId = "my-note-v1";
                    const unsub = db.collection('artifacts').doc(appId).collection('users').doc(user.uid).collection('data')
                        .onSnapshot(snap => {
                            setItems(snap.docs.map(doc => ({ id: doc.id, ...doc.data() })));
                            setLoading(false);
                        });
                    return () => unsub();
                });
            }, []);

            const saveToCloud = async (id, data) => {
                if (!dbRef.current || !userRef.current) return;
                const appId = "my-note-v1";
                await dbRef.current.collection('artifacts').doc(appId).collection('users').doc(userRef.current.uid).collection('data').doc(id).set(data);
            };

            const deleteFromCloud = async (id) => {
                if (!dbRef.current || !userRef.current) return;
                const appId = "my-note-v1";
                await dbRef.current.collection('artifacts').doc(appId).collection('users').doc(userRef.current.uid).collection('data').doc(id).delete();
            };

            const handleSubmit = async (e) => {
                e.preventDefault();
                const id = editingId || Date.now().toString();

                if (isBulkMode && activeTab === 'finance') {
                    const lines = content.split('\n').filter(l => l.trim());
                    for (let [i, line] of lines.entries()) {
                        const parts = line.split(/[:\-]/);
                        const bulkId = (Date.now() + i).toString();
                        await saveToCloud(bulkId, {
                            type: 'finance', title: parts[0]?.trim(), amount: parseFloat(parts[1]?.trim().replace(/[^0-9]/g, '') || '0'),
                            transactionType, date, createdAt: new Date().toISOString()
                        });
                    }
                    resetForm(); return;
                }

                await saveToCloud(id, {
                    type: activeTab, title, content, date,
                    amount: activeTab === 'finance' ? parseFloat(amount) || 0 : null,
                    transactionType: activeTab === 'finance' ? transactionType : null,
                    endDate: activeTab === 'subscribers' ? endDate || null : null,
                    createdAt: new Date().toISOString()
                });
                resetForm();
            };

            const resetForm = () => {
                setTitle(''); setContent(''); setAmount(''); setEndDate(''); setEditingId(null); setDate(new Date().toISOString().split('T')[0]);
            };

            const financeSummary = items
                .filter(i => i.type === 'finance' && new Date(i.date).getMonth() === new Date().getMonth())
                .reduce((acc, i) => {
                    if (i.transactionType === 'income') acc.income += i.amount; else acc.expense += i.amount;
                    return acc;
                }, { income: 0, expense: 0 });

            const filteredItems = items
                .filter(i => i.type === activeTab)
                .filter(i => i.title.toLowerCase().includes(searchTerm.toLowerCase()))
                .sort((a, b) => new Date(b.date) - new Date(a.date));

            if (loading) return <div className="h-screen flex items-center justify-center font-bold text-indigo-600 animate-pulse uppercase tracking-widest">Đang kết nối Server...</div>;

            return (
                <div className="flex h-screen overflow-hidden bg-slate-50">
                    <aside className={`fixed inset-y-0 left-0 z-50 w-64 bg-white border-r transform ${isMobileMenuOpen ? 'translate-x-0' : '-translate-x-full'} lg:relative lg:translate-x-0 transition-transform duration-300 flex flex-col`}>
                        <div className="p-6 border-b flex items-center gap-3">
                            <div className="w-10 h-10 bg-indigo-600 rounded-xl shadow-lg flex items-center justify-center text-white"><Icon name="cloud" size={24}/></div>
                            <span className="font-black text-xl tracking-tight">CloudNote</span>
                        </div>
                        <nav className="flex-1 p-4 space-y-1 overflow-y-auto custom-scrollbar">
                            {Object.entries(TABS).map(([key, info]) => (
                                <button key={key} onClick={() => { setActiveTab(key); setIsMobileMenuOpen(false); resetForm(); }} className={`w-full flex items-center gap-3 px-4 py-3 rounded-2xl font-bold transition-all ${activeTab === key ? 'bg-indigo-50 text-indigo-600 shadow-sm' : 'text-slate-400 hover:bg-slate-50 hover:text-slate-600'}`}>
                                    <Icon name={info.icon} size={20} /> {info.label}
                                </button>
                            ))}
                        </nav>
                        <div className="p-4 border-t text-center text-[10px] text-slate-400 font-bold uppercase tracking-widest">Server Online</div>
                    </aside>

                    <main className="flex-1 flex flex-col min-w-0">
                        <header className="h-16 bg-white/80 backdrop-blur border-b px-4 lg:px-8 flex items-center justify-between shrink-0">
                            <button className="lg:hidden p-2" onClick={() => setIsMobileMenuOpen(true)}><Icon name="menu" /></button>
                            <div className="relative w-full max-w-md mx-4">
                                <Icon name="search" className="absolute left-4 top-1/2 -translate-y-1/2 text-slate-400" size={16} />
                                <input type="text" placeholder="Tìm kiếm nhanh..." value={searchTerm} onChange={e => setSearchTerm(e.target.value)} className="w-full pl-12 pr-4 py-2 bg-slate-100 border-none rounded-full text-sm focus:ring-2 focus:ring-indigo-500 outline-none" />
                            </div>
                        </header>

                        <div className="flex-1 overflow-y-auto p-4 lg:p-8 custom-scrollbar">
                            <div className="max-w-4xl mx-auto space-y-8">
                                {activeTab === 'finance' && (
                                    <div className="grid grid-cols-1 md:grid-cols-3 gap-4 animate-in slide-in-from-top-2">
                                        <div className="bg-white p-5 rounded-3xl border border-slate-100 shadow-sm flex items-center gap-4">
                                            <div className="w-12 h-12 rounded-2xl bg-emerald-100 text-emerald-600 flex items-center justify-center"><Icon name="trending-up" /></div>
                                            <div><p className="text-[10px] font-black text-slate-400 uppercase">Thu nhập</p><p className="text-xl font-black text-emerald-600">{financeSummary.income.toLocaleString('vi-VN')}đ</p></div>
                                        </div>
                                        <div className="bg-white p-5 rounded-3xl border border-slate-100 shadow-sm flex items-center gap-4">
                                            <div className="w-12 h-12 rounded-2xl bg-rose-100 text-rose-600 flex items-center justify-center"><Icon name="trending-down" /></div>
                                            <div><p className="text-[10px] font-black text-slate-400 uppercase">Chi tiêu</p><p className="text-xl font-black text-rose-600">{financeSummary.expense.toLocaleString('vi-VN')}đ</p></div>
                                        </div>
                                        <div className="bg-indigo-600 p-5 rounded-3xl shadow-xl flex items-center gap-4 text-white">
                                            <div className="w-12 h-12 rounded-2xl bg-white/20 flex items-center justify-center"><Icon name="pie-chart" /></div>
                                            <div><p className="text-[10px] font-black text-white/60 uppercase">Dư tháng</p><p className="text-xl font-black">{(financeSummary.income - financeSummary.expense).toLocaleString('vi-VN')}đ</p></div>
                                        </div>
                                    </div>
                                )}

                                <div className="bg-white rounded-3xl border p-6 shadow-xl relative">
                                    <div className="flex justify-between items-center mb-6">
                                        <h2 className="font-black text-xl flex items-center gap-2 uppercase"><Icon name={editingId ? "edit-3" : "plus-circle"} /> {editingId ? "Cập nhật" : "Tạo mới"} {TABS[activeTab].label}</h2>
                                        {activeTab === 'finance' && !editingId && (
                                            <button onClick={() => setIsBulkMode(!isBulkMode)} className={`text-xs px-3 py-1.5 rounded-xl border font-bold ${isBulkMode ? 'bg-indigo-600 text-white' : 'text-slate-400'}`}>Nhập nhiều</button>
                                        )}
                                    </div>
                                    <form onSubmit={handleSubmit} className="space-y-4">
                                        <div className="grid grid-cols-1 md:grid-cols-12 gap-4">
                                            {!isBulkMode && (
                                                <div className="md:col-span-8">
                                                    <input type="text" placeholder="Tiêu đề..." value={title} onChange={e => setTitle(e.target.value)} className="w-full px-6 py-4 bg-slate-50 border-none rounded-2xl outline-none focus:ring-4 focus:ring-indigo-500/10 font-bold" required />
                                                </div>
                                            )}
                                            <div className={isBulkMode ? "md:col-span-12" : "md:col-span-4"}>
                                                <input type="date" value={date} onChange={e => setDate(e.target.value)} className="w-full px-6 py-4 bg-slate-50 border-none rounded-2xl outline-none font-bold" required />
                                            </div>
                                        </div>
                                        {activeTab === 'finance' && !isBulkMode && (
                                            <div className="grid grid-cols-1 md:grid-cols-12 gap-4">
                                                <div className="md:col-span-6"><input type="number" placeholder="Số tiền..." value={amount} onChange={e => setAmount(e.target.value)} className="w-full px-6 py-4 bg-slate-50 border-none rounded-2xl outline-none focus:ring-4 focus:ring-amber-500/10 font-black text-xl" required /></div>
                                                <div className="md:col-span-6 flex p-1 bg-slate-50 rounded-2xl gap-1">
                                                    <button type="button" onClick={() => setTransactionType('income')} className={`flex-1 py-3 rounded-xl text-xs font-black uppercase transition-all ${transactionType === 'income' ? 'bg-white text-emerald-600 shadow-sm' : 'text-slate-400'}`}>Thu</button>
                                                    <button type="button" onClick={() => setTransactionType('expense')} className={`flex-1 py-3 rounded-xl text-xs font-black uppercase transition-all ${transactionType === 'expense' ? 'bg-white text-rose-600 shadow-sm' : 'text-slate-400'}`}>Chi</button>
                                                </div>
                                            </div>
                                        )}
                                        <textarea placeholder={isBulkMode ? "Tên : Số tiền (Ví dụ Ăn cơm : 35000)" : "Nội dung chi tiết..."} value={content} onChange={e => setContent(e.target.value)} className="w-full px-6 py-4 bg-slate-50 border-none rounded-2xl outline-none min-h-[120px] font-medium" required />
                                        <div className="flex justify-end gap-3 pt-4">
                                            {editingId && <button type="button" onClick={resetForm} className="px-6 py-4 text-slate-400 font-bold">Hủy</button>}
                                            <button type="submit" className="px-12 py-4 bg-indigo-600 text-white font-black rounded-2xl shadow-xl shadow-indigo-100 hover:scale-[1.02] active:scale-95 transition-all uppercase tracking-widest">Lưu ngay</button>
                                        </div>
                                    </form>
                                </div>

                                <div className="space-y-4 pb-32">
                                    {filteredItems.map(item => (
                                        <div key={item.id} onClick={() => setExpandedItems(p => ({...p, [item.id]: !p[item.id]}))} className="bg-white p-6 rounded-[2rem] border border-white hover:border-indigo-200 transition-all cursor-pointer flex gap-5 group shadow-sm hover:shadow-xl">
                                            <div className={`w-14 h-14 rounded-2xl shrink-0 flex items-center justify-center ${item.transactionType === 'income' ? 'bg-emerald-100 text-emerald-600' : item.transactionType === 'expense' ? 'bg-rose-100 text-rose-600' : 'bg-slate-100 text-slate-400'}`}>
                                                <Icon name={TABS[item.type]?.icon || 'file'} size={28} />
                                            </div>
                                            <div className="flex-1 min-w-0">
                                                <div className="flex justify-between items-start">
                                                    <h4 className={`text-lg font-bold text-slate-800 ${expandedItems[item.id] ? '' : 'truncate pr-8'}`}>{item.title}</h4>
                                                    <span className="text-[10px] font-black text-slate-300 uppercase whitespace-nowrap ml-2">{new Date(item.date).toLocaleDateString('vi-VN')}</span>
                                                </div>
                                                {item.amount && <p className={`text-2xl font-black ${item.transactionType === 'income' ? 'text-emerald-600' : 'text-rose-600'}`}>{item.transactionType === 'income' ? '+' : '-'} {item.amount.toLocaleString('vi-VN')}đ</p>}
                                                <div className={`text-sm text-slate-500 font-medium mt-2 whitespace-pre-wrap ${expandedItems[item.id] ? '' : 'line-clamp-2'}`}>{item.content}</div>
                                                <div className={`flex gap-4 mt-5 transition-all ${expandedItems[item.id] ? 'opacity-100' : 'opacity-0 h-0 overflow-hidden'}`}>
                                                    <button onClick={(e) => { e.stopPropagation(); setEditingId(item.id); setTitle(item.title); setContent(item.content); setAmount(item.amount || ''); setDate(item.date); setTransactionType(item.transactionType || 'expense'); window.scrollTo({top:0, behavior:'smooth'}); }} className="text-[10px] font-black uppercase text-orange-400">Sửa</button>
                                                    <button onClick={(e) => { e.stopPropagation(); if(confirm('Xóa?')) deleteFromCloud(item.id); }} className="text-[10px] font-black uppercase text-rose-400">Xóa</button>
                                                </div>
                                            </div>
                                        </div>
                                    ))}
                                </div>
                            </div>
                        </div>
                    </main>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
