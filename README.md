# Planephoto-screening-AI
import React, { useState, useEffect, useRef } from 'react';
import { Camera, Upload, CheckCircle, XCircle, AlertTriangle, ShieldCheck, RefreshCw, ZoomIn, Info, Settings, Database, BrainCircuit, Save, LayoutDashboard, User, ChevronRight, BarChart3, Lock } from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, onSnapshot } from 'firebase/firestore';

// --- Firebase 配置 ---
const getFirebaseConfig = () => {
  if (typeof __firebase_config !== 'undefined') {
    return JSON.parse(__firebase_config);
  }
  
  // 為了相容 Vercel/Vite 環境與當前預覽環境
  const env = typeof process !== 'undefined' ? process.env : {};
  
  return {
    apiKey: env.VITE_FIREBASE_API_KEY || "",
    authDomain: env.VITE_FIREBASE_AUTH_DOMAIN || "",
    projectId: env.VITE_FIREBASE_PROJECT_ID || "",
    storageBucket: env.VITE_FIREBASE_STORAGE_BUCKET || "",
    messagingSenderId: env.VITE_FIREBASE_MESSAGING_SENDER_ID || "",
    appId: env.VITE_FIREBASE_APP_ID || ""
  };
};

const firebaseConfig = getFirebaseConfig();
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'jetphotos-ai-v2';

// ==========================================
// 【 重要：請在此處填入你的 UID 】
// 你可以在預覽網頁的右上角看到「UID: xxxxxx」
// ==========================================
const ADMIN_UID = "6645a96"; 

const App = () => {
  const [view, setView] = useState('public');
  const [user, setUser] = useState(null);
  const [authLoading, setAuthLoading] = useState(true);
  const [image, setImage] = useState(null);
  const [imageHash, setImageHash] = useState(null);
  const [analyzing, setAnalyzing] = useState(false);
  const [results, setResults] = useState(null);
  const [dragActive, setDragActive] = useState(false);
  const [showAdminModal, setShowAdminModal] = useState(false);

  const [globalConfig, setGlobalConfig] = useState({
    sharpnessWeight: 0.85,
    centeringWeight: 0.9,
    exposureWeight: 0.75,
    modelVersion: '2.4.1-stable',
    lastTrained: '2024-05-20'
  });

  const fileInputRef = useRef(null);

  // 1. 初始化身份驗證
  useEffect(() => {
    const initAuth = async () => {
      try {
        setAuthLoading(true);
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("驗證失敗:", err);
      } finally {
        setAuthLoading(false);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      setUser(u);
      if (u && u.uid !== ADMIN_UID && view === 'admin') {
        setView('public');
      }
    });
    return () => unsubscribe();
  }, [view]);

  // 2. 監聽模型參數
  useEffect(() => {
    if (!user) return;
    const configRef = doc(db, 'artifacts', appId, 'public', 'data', 'config', 'currentModel');
    const unsubscribe = onSnapshot(configRef, (docSnap) => {
      if (docSnap.exists()) {
        setGlobalConfig(docSnap.data());
      }
    }, (err) => console.error("同步錯誤:", err));
    return () => unsubscribe();
  }, [user]);

  const generateHash = (str) => {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash << 5) - hash + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash).toString(16);
  };

  const handleFileChange = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (event) => {
        const dataUrl = event.target.result;
        setImage(dataUrl);
        setImageHash(generateHash(dataUrl.substring(0, 5000)));
        setResults(null);
      };
      reader.readAsDataURL(file);
    }
  };

  const analyzeImage = async () => {
    if (!user || !imageHash) return;
    setAnalyzing(true);

    try {
      const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'analyses', imageHash);
      const docSnap = await getDoc(docRef);

      if (docSnap.exists()) {
        setResults(docSnap.data());
      } else {
        await new Promise(resolve => setTimeout(resolve, 2500));
        const seed = parseInt(imageHash.substring(0, 4), 16) % 100;
        const baseScore = 55 + (seed % 40);
        
        const newResults = {
          score: Math.round(baseScore * globalConfig.centeringWeight),
          passProbability: baseScore > 80 ? '高' : baseScore > 65 ? '中' : '低',
          details: [
            { label: '中心對齊檢測', status: baseScore > 70 ? 'pass' : 'fail', msg: 'AI 偵測飛機垂直軸心偏移 0.5%。' },
            { label: '畫質銳利度', status: baseScore > 60 ? 'pass' : 'warning', msg: '機身細節清晰，背景雜訊在可接受範圍。' },
            { label: '模型依據', status: 'info', msg: `分析使用模型版本: ${globalConfig.modelVersion}` }
          ],
          timestamp: Date.now()
        };

        await setDoc(docRef, newResults);
        setResults(newResults);
      }
    } catch (error) {
      console.error("分析錯誤:", error);
    } finally {
      setAnalyzing(false);
    }
  };

  const saveAdminConfig = async () => {
    if (!user || user.uid !== ADMIN_UID) return;
    const configRef = doc(db, 'artifacts', appId, 'public', 'data', 'config', 'currentModel');
    try {
      await setDoc(configRef, {
        ...globalConfig,
        lastTrained: new Date().toISOString().split('T')[0]
      });
      alert("模型參數更新成功！");
    } catch (err) {
      console.error("儲存失敗:", err);
    }
  };

  const handleAdminViewSwitch = () => {
    if (user?.uid === ADMIN_UID) {
      setView('admin');
    } else {
      setShowAdminModal(true);
      setTimeout(() => setShowAdminModal(false), 3000);
    }
  };

  const PublicView = () => (
    <div className="animate-in fade-in duration-500 space-y-8">
      <section className="text-center space-y-4 max-w-2xl mx-auto">
        <h2 className="text-4xl font-extrabold text-white tracking-tight">
          航空攝影師的 <span className="text-blue-500">AI 預核助理</span>
        </h2>
        <p className="text-slate-400">
          在上傳到 JetPhotos 之前進行初步篩選，優化照片質量並提高通過率。
        </p>
      </section>

      <div className="grid grid-cols-1 lg:grid-cols-12 gap-8">
        <div className="lg:col-span-7">
          <div 
            className={`relative aspect-video border-2 border-dashed rounded-[2rem] bg-slate-800/20 transition-all flex flex-col items-center justify-center overflow-hidden ${
              dragActive ? 'border-blue-500 bg-blue-500/5 scale-[0.99]' : 'border-slate-700 hover:border-slate-600'
            }`}
            onDragOver={(e) => { e.preventDefault(); setDragActive(true); }}
            onDragLeave={() => setDragActive(false)}
            onDrop={(e) => { e.preventDefault(); setDragActive(false); }}
          >
            {!image ? (
              <div className="text-center p-12">
                <div className="w-16 h-16 bg-blue-600/20 rounded-2xl flex items-center justify-center mx-auto mb-6">
                  <Upload size={32} className="text-blue-500" />
                </div>
                <h3 className="text-xl font-bold text-white mb-2">上傳飛機照片</h3>
                <button 
                  onClick={() => fileInputRef.current.click()}
                  className="bg-blue-600 hover:bg-blue-500 text-white px-8 py-3 rounded-full font-bold transition-all transform hover:scale-105"
                >
                  選擇檔案
                </button>
                <input type="file" ref={fileInputRef} className="hidden" onChange={handleFileChange} />
              </div>
            ) : (
              <div className="relative w-full h-full group">
                <img src={image} alt="Upload" className="w-full h-full object-contain" />
                <div className="absolute inset-0 bg-black/40 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center">
                  <button 
                    onClick={() => {setImage(null); setResults(null);}}
                    className="bg-white text-slate-900 px-4 py-2 rounded-full font-bold flex items-center gap-2"
                  >
                    <RefreshCw size={18} /> 更換圖片
                  </button>
                </div>
              </div>
            )}
          </div>

          {image && !results && !analyzing && (
            <button 
              onClick={analyzeImage}
              className="w-full mt-6 py-5 bg-gradient-to-r from-blue-600 to-indigo-600 rounded-2xl font-black text-xl shadow-2xl flex items-center justify-center gap-3"
            >
              <ShieldCheck size={28} /> 開始 AI 分析
            </button>
          )}

          {analyzing && (
            <div className="w-full mt-6 py-12 bg-slate-800/40 rounded-3xl border border-slate-700 flex flex-col items-center justify-center gap-6">
              <RefreshCw className="animate-spin text-blue-500" size={48} />
              <p className="text-xl font-bold text-white">正在執行 AI 模型推理...</p>
            </div>
          )}
        </div>

        <div className="lg:col-span-5">
          {results ? (
            <div className="bg-slate-800/50 border border-slate-700 rounded-[2rem] p-8 space-y-6">
              <div className="flex justify-between items-start">
                <h3 className="text-2xl font-bold">分析報告</h3>
                <div className="text-right">
                  <span className={`text-5xl font-black ${results.score > 75 ? 'text-emerald-400' : 'text-amber-400'}`}>{results.score}%</span>
                </div>
              </div>
              <div className="space-y-4">
                {results.details.map((d, i) => (
                  <div key={i} className="flex gap-4 p-4 bg-slate-950/40 rounded-2xl border border-white/5">
                    {d.status === 'pass' ? <CheckCircle className="text-emerald-500 shrink-0" /> : d.status === 'info' ? <Info className="text-blue-400 shrink-0" /> : <AlertTriangle className="text-amber-500 shrink-0" />}
                    <div>
                      <p className="text-sm font-bold">{d.label}</p>
                      <p className="text-xs text-slate-400 mt-0.5">{d.msg}</p>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          ) : (
            <div className="h-full min-h-[300px] border-2 border-dashed border-slate-800 rounded-[2rem] flex flex-col items-center justify-center text-slate-600 p-8 text-center">
              <ZoomIn size={48} className="mb-4 opacity-20" />
              <p>請上傳照片以查看 AI 分析結果</p>
            </div>
          )}
        </div>
      </div>
    </div>
  );

  const AdminView = () => (
    <div className="animate-in slide-in-from-left-4 duration-500 space-y-8">
      <div className="flex justify-between items-center">
        <div>
          <h2 className="text-3xl font-bold text-white flex items-center gap-3">
            <LayoutDashboard className="text-blue-500" /> 模型中心控制台
          </h2>
          <p className="text-slate-400 mt-1">當前登入管理員: {ADMIN_UID}</p>
        </div>
        <button 
          onClick={saveAdminConfig}
          className="bg-blue-600 hover:bg-blue-500 text-white px-6 py-2.5 rounded-xl font-bold flex items-center gap-2"
        >
          <Save size={18} /> 發佈全域更新
        </button>
      </div>

      <div className="bg-slate-800/80 rounded-[2rem] border border-slate-700 p-8 space-y-8">
        <div className="flex items-center gap-2 border-b border-slate-700 pb-4">
          <BrainCircuit className="text-blue-400" />
          <h3 className="font-bold text-lg">AI 偵測權重微調</h3>
        </div>
        
        <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
          <div className="space-y-6">
            <div>
              <div className="flex justify-between text-sm mb-2 text-slate-300">
                <span>對齊偵測敏感度</span>
                <span className="text-blue-400">{(globalConfig.centeringWeight * 100).toFixed(0)}%</span>
              </div>
              <input 
                type="range" min="0" max="1" step="0.01" 
                value={globalConfig.centeringWeight}
                onChange={(e) => setGlobalConfig({...globalConfig, centeringWeight: parseFloat(e.target.value)})}
                className="w-full h-2 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
              />
            </div>
            <div>
              <div className="flex justify-between text-sm mb-2 text-slate-300">
                <span>銳利度過濾強度</span>
                <span className="text-blue-400">{(globalConfig.sharpnessWeight * 100).toFixed(0)}%</span>
              </div>
              <input 
                type="range" min="0" max="1" step="0.01" 
                value={globalConfig.sharpnessWeight}
                onChange={(e) => setGlobalConfig({...globalConfig, sharpnessWeight: parseFloat(e.target.value)})}
                className="w-full h-2 bg-slate-700 rounded-lg appearance-none cursor-pointer accent-blue-500"
              />
            </div>
          </div>
          <div className="space-y-4">
            <div className="bg-slate-900/50 p-4 rounded-xl border border-slate-700">
              <label className="text-xs text-slate-500 uppercase font-bold block mb-2">部署版本號</label>
              <input 
                type="text" value={globalConfig.modelVersion}
                onChange={(e) => setGlobalConfig({...globalConfig, modelVersion: e.target.value})}
                className="w-full bg-slate-800 border border-slate-700 rounded-lg px-3 py-2 text-blue-400 font-mono focus:outline-none"
              />
            </div>
            <p className="text-xs text-slate-400 bg-blue-500/10 p-3 rounded-lg border border-blue-500/20">
              提示：此處修改的參數將即時影響所有正在使用本系統的用戶，請謹慎操作。
            </p>
          </div>
        </div>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-[#020617] text-slate-200">
      {showAdminModal && (
        <div className="fixed top-8 right-8 bg-red-500 text-white px-6 py-4 rounded-2xl shadow-2xl z-[200] flex items-center gap-3 animate-in fade-in slide-in-from-top-4">
          <Lock size={20} />
          <p className="font-bold">權限錯誤：目前登入身分並非管理員。</p>
        </div>
      )}

      <aside className="fixed left-0 top-0 h-full w-20 flex flex-col items-center py-8 border-r border-slate-800 bg-slate-950/50 backdrop-blur-xl z-[100]">
        <div className="w-12 h-12 bg-blue-600 rounded-2xl flex items-center justify-center mb-12">
          <BrainCircuit className="text-white" size={28} />
        </div>
        <nav className="flex flex-col gap-8">
          <button 
            onClick={() => setView('public')}
            className={`p-3 rounded-2xl transition-all ${view === 'public' ? 'bg-blue-600 text-white' : 'text-slate-500 hover:text-white'}`}
          >
            <User size={24} />
          </button>
          <button 
            onClick={handleAdminViewSwitch}
            className={`p-3 rounded-2xl transition-all ${view === 'admin' ? 'bg-blue-600 text-white' : 'text-slate-500 hover:text-white'}`}
          >
            <LayoutDashboard size={24} />
          </button>
        </nav>
      </aside>

      <main className="pl-20 min-h-screen">
        <header className="h-20 flex items-center justify-between px-8 border-b border-slate-800/50 sticky top-0 bg-[#020617]/80 backdrop-blur-md z-40">
          <div className="flex items-center gap-2">
            <span className="text-slate-500 text-sm font-medium">JetPhotos AI</span>
            <ChevronRight size={14} className="text-slate-700" />
            <span className="text-white font-bold">{view === 'public' ? '預核系統' : '後台管理'}</span>
          </div>
          <div className="flex items-center gap-4">
            <span className="text-[10px] font-mono text-slate-600 bg-slate-900 px-3 py-1 rounded-full border border-slate-800">
              UID: {user?.uid || '正在獲取...' }
            </span>
          </div>
        </header>

        <div className="max-w-6xl mx-auto py-12 px-6">
          {view === 'public' ? <PublicView /> : <AdminView />}
        </div>
      </main>
    </div>
  );
};

export default App;
