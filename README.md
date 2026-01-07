<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Personal Note App</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <!-- Babel for JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap');
        body { font-family: 'Inter', sans-serif; }
        .line-clamp-2 { display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; }
        .line-clamp-3 { display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }
    </style>
</head>
<body class="bg-slate-50">
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;
        
        // Mocking Lucide icons because we use CDN
        const Icon = ({ name, size = 20, strokeWidth = 2, className = "" }) => {
            useEffect(() => {
                lucide.createIcons();
            }, [name]);
            return <i data-lucide={name} style={{ width: size, height: size }} className={className}></i>;
        };

        const TABS = {
            finance: { id: 'finance', label: 'Thu Chi', icon: 'wallet', color: 'text-amber-600', bg: 'bg-amber-100' },
            todos: { id: 'todos', label: 'Việc Cần Làm', icon: 'check-square', color: 'text-orange-600', bg: 'bg-orange-100' },
            subscribers: { id: 'subscribers', label: 'Nhóm Khách Hàng', icon: 'users', color: 'text-pink-600', bg: 'bg-pink-100' },
            courses: { id: 'courses', label: 'Khóa Học', icon: 'book-open', color: 'text-blue-600', bg: 'bg-blue-100' },
            consulting: { id: 'consulting', label: 'Tư Vấn & Note', icon: 'message-circle', color: 'text-purple-600', bg: 'bg-purple-100' },
            links: { id: 'links', label: 'Kho Link', icon: 'link', color: 'text-emerald-600', bg: 'bg-emerald-100' },
        };

        const GROUP_COLORS = [
            { id: 'blue', bg: 'bg-blue-100', text: 'text-blue-700', border: 'border-blue-200' },
            { id: 'green', bg: 'bg-green-100', text: 'text-green-700', border: 'border-green-200' },
            { id: 'purple', bg: 'bg-purple-100', text: 'text-purple-700', border: 'border-purple-200' },
            { id: 'orange', bg: 'bg-orange-100', text: 'text-orange-700', border: 'border-orange-200' },
            { id: 'red', bg: 'bg-red-100', text: 'text-red-700', border: 'border-red-200' },
            { id: 'gray', bg: 'bg-gray-100', text: 'text-gray-700', border: 'border-gray-200' },
        ];

        function App() {
            const [activeTab, setActiveTab] = useState(() => localStorage.getItem('last_active_tab') || 'finance');
            const [items, setItems] = useState([]);
            const [searchTerm, setSearchTerm] = useState('');
            const [isMobileMenuOpen, setIsMobileMenuOpen] = useState(false);
            const [groups, setGroups] = useState([]);
            const [activeGroupId, setActiveGroupId] = useState('all'); 
            const [isAddingGroup, setIsAddingGroup] = useState(false);
            const [newGroupName, setNewGroupName] = useState('');
            const [newGroupColor, setNewGroupColor] = useState('blue');
            const [confirmDelete, setConfirmDelete] = useState({ show: false, type: '', id: null, name: '' });
            const [editingId, setEditingId] = useState(null); 
            const [isBulkMode, setIsBulkMode] = useState(false); 
            const [isChecklistMode, setIsChecklistMode] = useState(false);
            const [title, setTitle] = useState('');
            const [content, setContent] = useState('');
            const [url, setUrl] = useState('');
            const [itemGroupId, setItemGroupId] = useState(''); 
            const [amount, setAmount] = useState('');
            const [transactionType, setTransactionType] = useState('expense'); 
            const [date, setDate] = useState(new Date().toISOString().split('T')[0]); 
            const [endDate, setEndDate] = useState(''); 
            const [expandedItems, setExpandedItems] = useState({}); 
            const [copiedId, setCopiedId] = useState(null);
            const fileInputRef = useRef(null);

            useEffect(() => {
                const savedData = localStorage.getItem('my_personal_notes_v3');
                if (savedData) setItems(JSON.parse(savedData));
                const savedGroups = localStorage.getItem('my_personal_notes_groups');
                if (savedGroups) setGroups(JSON.parse(savedGroups));
                else setGroups([
                    { id: 'eat', name: 'Ăn uống', color: 'orange', type: 'finance' },
                    { id: 'move', name: 'Di chuyển', color: 'blue', type: 'finance' },
                    { id: 'vip', name: 'Khách VIP', color: 'purple', type: 'subscribers' }
                ]);
            }, []);

            useEffect(() => { localStorage.setItem('my_personal_notes_v3', JSON.stringify(items)); }, [items]);
            useEffect(() => { localStorage.setItem('my_personal_notes_groups', JSON.stringify(groups)); }, [groups]);
            useEffect(() => { localStorage.setItem('last_active_tab', activeTab); }, [activeTab]);

            const formatCurrency = (num) => new Intl.NumberFormat('vi-VN', { style: 'currency', currency: 'VND' }).format(num);
            const getColorClass = (colorId) => GROUP_COLORS.find(c => c.id === colorId) || GROUP_COLORS[0];
            const toggleExpand = (id) => setExpandedItems(prev => ({ ...prev, [id]: !prev[id] }));

            const copyToClipboard = (text, id) => {
                const textArea = document.createElement("textarea");
                textArea.value = text;
                document.body.appendChild(textArea);
                textArea.select();
                document.execCommand('copy');
                document.body.removeChild(textArea);
                setCopiedId(id);
                setTimeout(() => setCopiedId(null), 2000);
            };

            const handleSubmit = (e) => {
                e.preventDefault();
                const baseData = { type: activeTab, title, content, date: date || new Date().toISOString().split('T')[0], createdAt: new Date().toISOString(), groupId: itemGroupId || null };
                
                if (activeTab === 'subscribers') {
                    baseData.endDate = endDate || null;
                } else if (activeTab === 'finance') {
                    if (isBulkMode) {
                        const lines = content.split('\n').filter(l => l.trim());
                        const newFinance = lines.map((l, i) => {
                            const p = l.split(/[:\-]/);
                            return { id: Date.now() + i, type: 'finance', title: p[0]?.trim(), amount: parseFloat(p[1]?.trim().replace(/[^0-9]/g, '') || '0'), transactionType, date, groupId: itemGroupId, createdAt: new Date().toISOString() };
                        });
                        setItems(prev => [...newFinance, ...prev]);
                        resetForm(); return;
                    }
                    baseData.amount = parseFloat(amount) || 0;
                    baseData.transactionType = transactionType;
                } else if (activeTab === 'todos') {
                    if (isBulkMode) {
                        const lines = content.split('\n').filter(l => l.trim());
                        setItems(prev => [...lines.map((l, i) => ({ id: Date.now() + i, type: 'todos', title: l.trim(), date, completed: false, createdAt: new Date().toISOString() })), ...prev]);
                        resetForm(); return;
                    }
                }

                if (editingId) setItems(prev => prev.map(i => i.id === editingId ? { ...i, ...baseData } : i));
                else setItems(prev => [{ id: Date.now(), ...baseData, completed: false }, ...prev]);
                resetForm();
            };

            const resetForm = () => {
                setTitle(''); setContent(''); setUrl(''); setEndDate(''); setAmount(''); setItemGroupId('');
                setEditingId(null); setIsBulkMode(false);
            };

            const groupMonthlyTotal = items
                .filter(i => i.type === 'finance' && (new Date(i.date).getMonth() === new Date().getMonth()) && (activeGroupId === 'all' ? true : i.groupId === activeGroupId))
                .reduce((acc, i) => {
                    if (i.transactionType === 'income') acc.income += i.amount; else acc.expense += i.amount;
                    return acc;
                }, { income: 0, expense: 0 });

            const filteredItems = items
                .filter(i => i.type === activeTab)
                .filter(i => (activeTab === 'subscribers' || activeTab === 'finance') && activeGroupId !== 'all' ? i.groupId === activeGroupId : true)
                .filter(i => i.title.toLowerCase().includes(searchTerm.toLowerCase()))
                .sort((a, b) => new Date(b.date) - new Date(a.date));

            return (
                <div className="flex h-screen overflow-hidden">
                    {/* Sidebar */}
                    <aside className={`fixed inset-y-0 left-0 z-50 w-64 bg-white border-r transform ${isMobileMenuOpen ? 'translate-x-0' : '-translate-x-full'} lg:relative lg:translate-x-0 transition-transform duration-200 flex flex-col`}>
                        <div className="p-6 border-b font-bold text-xl flex items-center gap-2">
                            <div className="w-8 h-8 bg-indigo-600 rounded shadow-lg"></div> My Notes
                        </div>
                        <nav className="flex-1 p-4 space-y-1 overflow-y-auto">
                            {Object.entries(TABS).map(([key, info]) => (
                                <button key={key} onClick={() => { setActiveTab(key); setIsMobileMenuOpen(false); resetForm(); }} className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl font-medium ${activeTab === key ? 'bg-slate-100 text-indigo-600 shadow-sm' : 'text-slate-500 hover:bg-slate-50'}`}>
                                    <Icon name={info.icon} size={18} /> {info.label}
                                </button>
                            ))}
                        </nav>
                    </aside>

                    {/* Main */}
                    <main className="flex-1 flex flex-col min-w-0">
                        <header className="h-16 bg-white border-b px-4 lg:px-8 flex items-center justify-between shrink-0">
                            <button className="lg:hidden p-2" onClick={() => setIsMobileMenuOpen(true)}><Icon name="menu" /></button>
                            <div className="relative w-full max-w-md ml-4 lg:ml-0">
                                <Icon name="search" className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" size={16} />
                                <input type="text" placeholder="Tìm kiếm nhanh..." value={searchTerm} onChange={e => setSearchTerm(e.target.value)} className="w-full pl-10 pr-4 py-2 bg-slate-100 border-none rounded-full text-sm focus:ring-2 focus:ring-indigo-500 outline-none" />
                            </div>
                        </header>

                        <div className="flex-1 overflow-y-auto p-4 lg:p-8 space-y-6">
                            <div className="max-w-4xl mx-auto space-y-6">
                                {/* Finance Stats */}
                                {activeTab === 'finance' && (
                                    <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                                        <div className="bg-white p-4 rounded-2xl border shadow-sm flex items-center gap-4">
                                            <div className="w-10 h-10 rounded-xl bg-emerald-100 text-emerald-600 flex items-center justify-center"><Icon name="trending-up" /></div>
                                            <div><p className="text-[10px] font-bold text-slate-400 uppercase">Thu Tháng</p><p className="text-lg font-bold text-emerald-600">{formatCurrency(groupMonthlyTotal.income)}</p></div>
                                        </div>
                                        <div className="bg-white p-4 rounded-2xl border shadow-sm flex items-center gap-4">
                                            <div className="w-10 h-10 rounded-xl bg-red-100 text-red-600 flex items-center justify-center"><Icon name="trending-down" /></div>
                                            <div><p className="text-[10px] font-bold text-slate-400 uppercase">Chi Tháng</p><p className="text-lg font-bold text-red-600">{formatCurrency(groupMonthlyTotal.expense)}</p></div>
                                        </div>
                                        <div className="bg-indigo-600 p-4 rounded-2xl shadow-lg flex items-center gap-4 text-white">
                                            <div className="w-10 h-10 rounded-xl bg-white/20 flex items-center justify-center"><Icon name="pie-chart" /></div>
                                            <div><p className="text-[10px] font-bold text-white/60 uppercase">Dư Tháng</p><p className="text-lg font-bold">{formatCurrency(groupMonthlyTotal.income - groupMonthlyTotal.expense)}</p></div>
                                        </div>
                                    </div>
                                )}

                                {/* Form */}
                                <div className="bg-white rounded-2xl border p-6 shadow-sm">
                                    <div className="flex justify-between items-center mb-4">
                                        <h2 className="font-bold text-lg flex items-center gap-2"><Icon name={editingId ? "edit-2" : "plus"} /> {editingId ? "Sửa" : "Thêm mới"} {TABS[activeTab].label}</h2>
                                        {(activeTab === 'finance' || activeTab === 'todos') && !editingId && (
                                            <button onClick={() => setIsBulkMode(!isBulkMode)} className={`text-xs px-3 py-1 rounded-lg border font-bold ${isBulkMode ? 'bg-indigo-600 text-white' : 'text-slate-500'}`}>Nhập nhiều</button>
                                        )}
                                    </div>
                                    <form onSubmit={handleSubmit} className="space-y-4">
                                        <div className="grid grid-cols-1 md:grid-cols-12 gap-4">
                                            {!isBulkMode && (
                                                <div className="md:col-span-8">
                                                    <input type="text" placeholder="Tiêu đề..." value={title} onChange={e => setTitle(e.target.value)} className="w-full px-4 py-2 bg-slate-50 rounded-xl outline-none focus:ring-2 focus:ring-indigo-500" required />
                                                </div>
                                            )}
                                            <div className={isBulkMode ? "md:col-span-12" : "md:col-span-4"}>
                                                <input type="date" value={date} onChange={e => setDate(e.target.value)} className="w-full px-4 py-2 bg-slate-50 rounded-xl outline-none" required />
                                            </div>
                                        </div>
                                        {activeTab === 'finance' && (
                                            <div className="grid grid-cols-1 md:grid-cols-12 gap-4">
                                                {!isBulkMode && (
                                                    <div className="md:col-span-6"><input type="number" placeholder="Số tiền..." value={amount} onChange={e => setAmount(e.target.value)} className="w-full px-4 py-2 bg-slate-50 rounded-xl outline-none" /></div>
                                                )}
                                                <div className="md:col-span-6 flex gap-2">
                                                    <button type="button" onClick={() => setTransactionType('income')} className={`flex-1 py-2 rounded-xl text-xs font-bold ${transactionType === 'income' ? 'bg-emerald-500 text-white' : 'bg-slate-50 text-slate-400'}`}>Thu nhập</button>
                                                    <button type="button" onClick={() => setTransactionType('expense')} className={`flex-1 py-2 rounded-xl text-xs font-bold ${transactionType === 'expense' ? 'bg-red-500 text-white' : 'bg-slate-50 text-slate-400'}`}>Chi phí</button>
                                                </div>
                                            </div>
                                        )}
                                        <textarea placeholder={isBulkMode && activeTab === 'finance' ? "Tên : Số tiền (ví dụ Ăn trưa : 50000)" : "Nội dung chi tiết..."} value={content} onChange={e => setContent(e.target.value)} className="w-full px-4 py-2 bg-slate-50 rounded-xl outline-none min-h-[100px]" />
                                        <div className="flex justify-end gap-2">
                                            {editingId && <button type="button" onClick={resetForm} className="px-4 py-2 text-slate-500">Hủy</button>}
                                            <button type="submit" className="px-8 py-2 bg-indigo-600 text-white font-bold rounded-xl shadow-lg">Lưu lại</button>
                                        </div>
                                    </form>
                                </div>

                                {/* List */}
                                <div className="space-y-4 pb-20">
                                    {filteredItems.map(item => (
                                        <div key={item.id} onClick={() => toggleExpand(item.id)} className="bg-white p-5 rounded-2xl border hover:border-indigo-300 transition-all cursor-pointer flex gap-4">
                                            <div className={`w-12 h-12 rounded-xl shrink-0 flex items-center justify-center ${item.transactionType === 'income' ? 'bg-emerald-100 text-emerald-600' : item.transactionType === 'expense' ? 'bg-red-100 text-red-600' : 'bg-slate-100 text-slate-500'}`}>
                                                <Icon name={TABS[item.type]?.icon || 'file'} />
                                            </div>
                                            <div className="flex-1 min-w-0">
                                                <div className="flex justify-between items-start">
                                                    <h4 className={`font-bold truncate pr-4 ${expandedItems[item.id] ? 'whitespace-normal' : ''}`}>{item.title}</h4>
                                                    <span className="text-[10px] font-bold text-slate-400">{new Date(item.date).toLocaleDateString('vi-VN')}</span>
                                                </div>
                                                {item.amount && <p className={`text-lg font-black ${item.transactionType === 'income' ? 'text-emerald-600' : 'text-red-600'}`}>{item.transactionType === 'income' ? '+' : '-'} {formatCurrency(item.amount)}</p>}
                                                <p className={`text-sm text-slate-500 mt-1 whitespace-pre-wrap ${expandedItems[item.id] ? '' : 'line-clamp-2'}`}>{item.content}</p>
                                                <div className="flex gap-2 mt-3 opacity-0 group-hover:opacity-100 transition-opacity">
                                                    <button onClick={(e) => { e.stopPropagation(); handleEdit(item); }} className="text-xs font-bold text-orange-500">Sửa</button>
                                                    <button onClick={(e) => { e.stopPropagation(); if(confirm('Xóa?')) setItems(prev => prev.filter(i => i.id !== item.id)); }} className="text-xs font-bold text-red-500">Xóa</button>
                                                    <button onClick={(e) => { e.stopPropagation(); copyToClipboard(item.content || item.title, item.id); }} className="text-xs font-bold text-indigo-500">{copiedId === item.id ? 'Đã copy' : 'Copy'}</button>
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
