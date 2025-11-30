import React, { useState, useEffect, useRef } from 'react';
import { Play, Pause, RotateCcw, CheckCircle, Plus, Trash2, Volume2, VolumeX, Coffee, Brain, Armchair } from 'lucide-react';

const PomodoroTimer = () => {
  // タイマー設定 (分)
  const MODES = {
    focus: { label: '集中', minutes: 25, color: 'bg-rose-500', icon: <Brain size={20} /> },
    short: { label: '短休憩', minutes: 5, color: 'bg-teal-500', icon: <Coffee size={20} /> },
    long: { label: '長休憩', minutes: 15, color: 'bg-indigo-500', icon: <Armchair size={20} /> },
  };

  const [mode, setMode] = useState('focus');
  const [timeLeft, setTimeLeft] = useState(MODES.focus.minutes * 60);
  const [isActive, setIsActive] = useState(false);
  const [tasks, setTasks] = useState([]);
  const [newTask, setNewTask] = useState('');
  const [soundEnabled, setSoundEnabled] = useState(true);
  
  // AudioContextのリファレンス（再利用のため）
  const audioContextRef = useRef(null);

  // タイマーロジック
  useEffect(() => {
    let interval = null;

    if (isActive && timeLeft > 0) {
      interval = setInterval(() => {
        setTimeLeft((prevTime) => prevTime - 1);
      }, 1000);
    } else if (timeLeft === 0) {
      setIsActive(false);
      if (soundEnabled) playNotificationSound();
    }

    return () => clearInterval(interval);
  }, [isActive, timeLeft, soundEnabled]);

  // モード切り替え
  const switchMode = (newMode) => {
    setMode(newMode);
    setTimeLeft(MODES[newMode].minutes * 60);
    setIsActive(false);
  };

  // タイマー操作
  const toggleTimer = () => setIsActive(!isActive);
  
  const resetTimer = () => {
    setIsActive(false);
    setTimeLeft(MODES[mode].minutes * 60);
  };

  // 時間フォーマット (MM:SS)
  const formatTime = (seconds) => {
    const mins = Math.floor(seconds / 60);
    const secs = seconds % 60;
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  // 進行状況の計算（プログレスバー用）
  const progress = ((MODES[mode].minutes * 60 - timeLeft) / (MODES[mode].minutes * 60)) * 100;

  // タスク管理
  const addTask = (e) => {
    e.preventDefault();
    if (!newTask.trim()) return;
    setTasks([...tasks, { id: Date.now(), text: newTask, completed: false }]);
    setNewTask('');
  };

  const toggleTask = (id) => {
    setTasks(tasks.map(task => 
      task.id === id ? { ...task, completed: !task.completed } : task
    ));
  };

  const deleteTask = (id) => {
    setTasks(tasks.filter(task => task.id !== id));
  };

  // Web Audio APIを使用した通知音
  const playNotificationSound = () => {
    try {
      if (!audioContextRef.current) {
        audioContextRef.current = new (window.AudioContext || window.webkitAudioContext)();
      }
      const ctx = audioContextRef.current;
      const oscillator = ctx.createOscillator();
      const gainNode = ctx.createGain();

      oscillator.connect(gainNode);
      gainNode.connect(ctx.destination);

      oscillator.type = 'sine';
      oscillator.frequency.setValueAtTime(880, ctx.currentTime); // A5
      oscillator.frequency.exponentialRampToValueAtTime(440, ctx.currentTime + 0.5); // 下がる音
      
      gainNode.gain.setValueAtTime(0.3, ctx.currentTime);
      gainNode.gain.exponentialRampToValueAtTime(0.01, ctx.currentTime + 0.5);

      oscillator.start();
      oscillator.stop(ctx.currentTime + 0.5);
    } catch (error) {
      console.error("Audio play failed", error);
    }
  };

  // 現在のテーマカラーを取得
  const currentTheme = MODES[mode].color;
  const textColor = mode === 'focus' ? 'text-rose-600' : mode === 'short' ? 'text-teal-600' : 'text-indigo-600';
  const lightBgColor = mode === 'focus' ? 'bg-rose-50' : mode === 'short' ? 'bg-teal-50' : 'bg-indigo-50';

  return (
    <div className={`min-h-screen transition-colors duration-500 ease-in-out ${currentTheme} p-4 md:p-8 font-sans`}>
      <div className="max-w-md mx-auto bg-white/95 backdrop-blur rounded-3xl shadow-2xl overflow-hidden">
        
        {/* ヘッダー */}
        <div className="p-6 text-center border-b border-gray-100 relative">
          <h1 className="text-2xl font-bold text-gray-800 flex items-center justify-center gap-2">
            ポモドーロタイマー
          </h1>
          <button 
            onClick={() => setSoundEnabled(!soundEnabled)}
            className="absolute right-6 top-7 text-gray-400 hover:text-gray-600 transition-colors"
          >
            {soundEnabled ? <Volume2 size={20} /> : <VolumeX size={20} />}
          </button>
        </div>

        {/* モード切り替えタブ */}
        <div className="flex p-2 bg-gray-50 m-4 rounded-xl">
          {Object.keys(MODES).map((m) => (
            <button
              key={m}
              onClick={() => switchMode(m)}
              className={`flex-1 py-2 rounded-lg text-sm font-medium transition-all duration-300 flex items-center justify-center gap-2 ${
                mode === m 
                  ? 'bg-white text-gray-800 shadow-sm' 
                  : 'text-gray-500 hover:bg-gray-100'
              }`}
            >
              {MODES[m].icon}
              <span className="hidden sm:inline">{MODES[m].label}</span>
            </button>
          ))}
        </div>

        {/* タイマー表示エリア */}
        <div className="py-12 text-center relative">
          {/* プログレスリング背景 (簡易的) */}
          <div className="absolute top-0 left-0 h-1 bg-gray-100 w-full">
            <div 
              className={`h-full transition-all duration-1000 ${currentTheme}`} 
              style={{ width: `${progress}%` }}
            />
          </div>

          <div className={`text-8xl font-bold tabular-nums tracking-tight ${textColor} transition-colors duration-500`}>
            {formatTime(timeLeft)}
          </div>
          <p className="text-gray-500 mt-2 font-medium">
            {isActive ? (mode === 'focus' ? '集中しましょう！' : 'リラックスタイム') : '準備はいいですか？'}
          </p>

          {/* コントロールボタン */}
          <div className="flex justify-center gap-4 mt-8">
            <button
              onClick={toggleTimer}
              className={`h-16 w-32 rounded-2xl flex items-center justify-center gap-2 text-white font-bold text-lg shadow-lg transform active:scale-95 transition-all duration-300 ${currentTheme} hover:opacity-90`}
            >
              {isActive ? <Pause fill="currentColor" /> : <Play fill="currentColor" />}
              {isActive ? '一時停止' : '開始'}
            </button>
            
            <button
              onClick={resetTimer}
              className="h-16 w-16 rounded-2xl bg-gray-100 text-gray-500 flex items-center justify-center hover:bg-gray-200 transition-colors active:scale-95"
              title="リセット"
            >
              <RotateCcw size={24} />
            </button>
          </div>
        </div>

        {/* タスクセクション */}
        <div className="bg-gray-50 p-6 rounded-t-3xl min-h-[300px]">
          <h2 className="text-lg font-bold text-gray-700 mb-4 flex items-center gap-2">
            タスクリスト
            <span className="text-xs bg-gray-200 text-gray-600 px-2 py-1 rounded-full">{tasks.filter(t => !t.completed).length}</span>
          </h2>
          
          <form onSubmit={addTask} className="flex gap-2 mb-4">
            <input
              type="text"
              value={newTask}
              onChange={(e) => setNewTask(e.target.value)}
              placeholder="何に取り組みますか？"
              className="flex-1 px-4 py-3 rounded-xl border-none shadow-sm focus:ring-2 focus:ring-rose-200 outline-none"
            />
            <button 
              type="submit"
              className={`p-3 rounded-xl text-white shadow-sm transition-colors ${currentTheme} hover:opacity-90`}
            >
              <Plus size={24} />
            </button>
          </form>

          <div className="space-y-2 max-h-60 overflow-y-auto pr-2 custom-scrollbar">
            {tasks.length === 0 && (
              <div className="text-center py-8 text-gray-400 text-sm">
                タスクがありません。<br/>新しいタスクを追加して作業を始めましょう。
              </div>
            )}
            {tasks.map((task) => (
              <div 
                key={task.id} 
                className={`group flex items-center justify-between p-3 rounded-xl bg-white shadow-sm border border-transparent transition-all ${
                  task.completed ? 'opacity-60 bg-gray-50' : 'hover:border-gray-200'
                }`}
              >
                <div className="flex items-center gap-3 overflow-hidden">
                  <button
                    onClick={() => toggleTask(task.id)}
                    className={`flex-shrink-0 transition-colors ${
                      task.completed ? 'text-green-500' : 'text-gray-300 hover:text-rose-400'
                    }`}
                  >
                    <CheckCircle size={22} fill={task.completed ? "currentColor" : "none"} />
                  </button>
                  <span className={`truncate text-gray-700 ${task.completed ? 'line-through text-gray-400' : ''}`}>
                    {task.text}
                  </span>
                </div>
                <button
                  onClick={() => deleteTask(task.id)}
                  className="text-gray-300 hover:text-red-400 opacity-0 group-hover:opacity-100 transition-opacity p-1"
                >
                  <Trash2 size={18} />
                </button>
              </div>
            ))}
          </div>
        </div>
      </div>
      
      <div className="text-center text-white/80 mt-6 text-sm">
        Stay Focused. Be Productive.
      </div>

      <style jsx>{`
        .custom-scrollbar::-webkit-scrollbar {
          width: 6px;
        }
        .custom-scrollbar::-webkit-scrollbar-track {
          background: transparent;
        }
        .custom-scrollbar::-webkit-scrollbar-thumb {
          background-color: rgba(156, 163, 175, 0.3);
          border-radius: 20px;
        }
      `}</style>
    </div>
  );
};

export default PomodoroTimer;
