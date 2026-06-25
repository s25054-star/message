[Uploading code_artifact.tsx…]()
# messageimport React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot } from 'firebase/firestore';
import { Send, LogOut, KeyRound, User as UserIcon, MessageCircle } from 'lucide-react';

// Firebaseの初期化（裏側のデータベース接続）
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

export default function App() {
  // アプリの状態（データ）を管理
  const [user, setUser] = useState(null);
  const [roomCode, setRoomCode] = useState('');
  const [userName, setUserName] = useState('');
  const [isJoined, setIsJoined] = useState(false);
  const [messages, setMessages] = useState([]);
  const [inputText, setInputText] = useState('');
  const [error, setError] = useState('');
  
  // チャットの自動スクロール用
  const messagesEndRef = useRef(null);

  // メッセージが追加されたら一番下までスクロールする
  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  };
  useEffect(() => {
    scrollToBottom();
  }, [messages]);

  // 【ステップ1】ユーザーの匿名ログイン処理
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("認証エラー:", err);
      }
    };
    initAuth();

    // ログイン状態の監視
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  // 【ステップ2】チャットルームのメッセージをリアルタイムで取得する処理
  useEffect(() => {
    // ログインしていない、または入室していない場合は何もしない
    if (!user || !isJoined || !roomCode) return;

    // Firebaseのパス（/ や \ はエラーになるため _ に置換）
    const safeRoomCode = roomCode.trim().replace(/[\/\\]/g, '_');
    const collectionName = `chat_room_${safeRoomCode}`;
    const messagesRef = collection(db, 'artifacts', appId, 'public', 'data', collectionName);
    
    // データベースの変更を監視して画面に反映
    const unsubscribe = onSnapshot(messagesRef, (snapshot) => {
      const msgs = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      // 時間順（古い順）に並び替え
      msgs.sort((a, b) => a.timestamp - b.timestamp);
      setMessages(msgs);
    }, (error) => {
      console.error("メッセージ取得エラー:", error);
      setError("メッセージの読み込みに失敗しました。");
    });

    return () => unsubscribe();
  }, [user, isJoined, roomCode]);

  // 入室ボタンを押した時の処理
  const handleJoin = (e) => {
    e.preventDefault();
    if (!userName.trim()) {
      setError("あなたの名前を入力してください。");
      return;
    }
    if (!roomCode.trim()) {
      setError("合言葉（ルームコード）を入力してください。");
      return;
    }
    setError('');
    setIsJoined(true);
  };

  // メッセージ送信処理
  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!inputText.trim() || !user) return;

    const safeRoomCode = roomCode.trim().replace(/[\/\\]/g, '_');
    const collectionName = `chat_room_${safeRoomCode}`;
    const messagesRef = collection(db, 'artifacts', appId, 'public', 'data', collectionName);
    
    const msgData = {
      text: inputText,
      senderId: user.uid,
      senderName: userName.trim(),
      timestamp: Date.now()
    };
    
    setInputText(''); // 送信したら入力欄を空にする
    try {
      await addDoc(messagesRef, msgData);
    } catch (err) {
      console.error("送信エラー:", err);
      setError("メッセージの送信に失敗しました。");
    }
  };

  // 時間を「14:05」のような形式にフォーマットする関数
  const formatTime = (ts) => {
    const d = new Date(ts);
    return `${d.getHours().toString().padStart(2, '0')}:${d.getMinutes().toString().padStart(2, '0')}`;
  };

  // ログイン処理中のローディング画面
  if (!user) {
    return (
      <div className="flex h-screen items-center justify-center bg-gray-100 text-gray-500 font-medium">
        接続中...
      </div>
    );
  }

  // ■■■ 入室画面（セットアップ） ■■■
  if (!isJoined) {
    return (
      <div className="bg-gray-100 min-h-screen flex items-center justify-center sm:p-4">
        <div className="bg-white w-full max-w-md h-screen sm:h-auto sm:rounded-3xl shadow-xl overflow-hidden flex flex-col p-8 space-y-8 justify-center">
          <div className="text-center space-y-3">
            <div className="bg-green-100 w-20 h-20 rounded-full flex items-center justify-center mx-auto mb-2 shadow-inner">
              <MessageCircle className="text-green-500" size={40} />
            </div>
            <h2 className="text-2xl font-extrabold text-gray-800">秘密のチャット</h2>
            <p className="text-gray-500 text-sm leading-relaxed">
              合言葉を知っている人同士で<br />安全にリアルタイムチャットができます。
            </p>
          </div>

          <form onSubmit={handleJoin} className="space-y-5">
            {error && (
              <div className="bg-red-50 text-red-600 p-3 rounded-lg text-sm text-center font-bold">
                {error}
              </div>
            )}
            
            <div className="space-y-2">
              <label className="text-sm font-bold text-gray-700 flex items-center space-x-1.5">
                <UserIcon size={18} className="text-green-500" /> <span>あなたの名前</span>
              </label>
              <input 
                type="text" 
                value={userName}
                onChange={e => setUserName(e.target.value)}
                className="w-full border-2 border-gray-200 rounded-xl px-4 py-3 focus:outline-none focus:border-green-400 focus:ring-2 focus:ring-green-100 transition"
                placeholder="例：タロウ"
                maxLength={20}
              />
            </div>

            <div className="space-y-2">
              <label className="text-sm font-bold text-gray-700 flex items-center space-x-1.5">
                <KeyRound size={18} className="text-green-500" /> <span>合言葉（ルームコード）</span>
              </label>
              <input 
                type="text" 
                value={roomCode}
                onChange={e => setRoomCode(e.target.value)}
                className="w-full border-2 border-gray-200 rounded-xl px-4 py-3 focus:outline-none focus:border-green-400 focus:ring-2 focus:ring-green-100 font-mono transition text-lg"
                placeholder="例：secret123"
              />
            </div>

            <button 
              type="submit"
              className="w-full bg-green-500 text-white font-bold text-lg py-4 rounded-xl hover:bg-green-600 active:bg-green-700 transition shadow-md mt-6"
            >
              チャットをはじめる
            </button>
          </form>
        </div>
      </div>
    );
  }

  // ■■■ チャット画面 ■■■
  return (
    <div className="bg-gray-100 min-h-screen flex items-center justify-center sm:p-4">
      {/* スマホ風のコンテナ */}
      <div className="bg-[#7494a3] w-full max-w-md h-screen sm:h-[850px] sm:max-h-[90vh] sm:rounded-[2.5rem] sm:shadow-2xl overflow-hidden flex flex-col relative sm:border-[8px] border-gray-800">
        
        {/* ヘッダー */}
        <header className="bg-gray-800/90 backdrop-blur-sm text-white p-4 flex justify-between items-center z-10">
          <div className="flex items-center space-x-2 truncate">
            <MessageCircle size={20} className="text-green-400 flex-shrink-0" />
            <h1 className="font-bold text-md truncate pr-2">ルーム: {roomCode}</h1>
          </div>
          <button 
            onClick={() => setIsJoined(false)} 
            className="flex items-center space-x-1 text-xs bg-gray-700 px-3 py-1.5 rounded-full hover:bg-gray-600 transition flex-shrink-0"
          >
            <LogOut size={14} />
            <span>退出</span>
          </button>
        </header>

        {/* メッセージ一覧エリア */}
        <div className="flex-1 overflow-y-auto p-4 space-y-4 pb-4">
          {messages.length === 0 && (
            <div className="text-center text-white/70 text-sm mt-10 bg-black/10 rounded-xl p-4 w-fit mx-auto">
              まだメッセージはありません。<br/>挨拶を送ってみましょう！
            </div>
          )}

          {messages.map(msg => {
            const isMine = msg.senderId === user.uid;
            return (
              <div key={msg.id} className={`flex flex-col ${isMine ? 'items-end' : 'items-start'}`}>
                {/* 送信者の名前（自分と相手の両方を表示） */}
                <span className="text-xs font-bold text-white/90 mb-1 mx-1 drop-shadow-md">
                  {msg.senderName} {isMine && <span className="font-normal opacity-75 text-[10px]">(あなた)</span>}
                </span>
                
                <div className="flex items-end space-x-1.5">
                  {/* 自分のメッセージの時間 */}
                  {isMine && <span className="text-[10px] text-white/80 mb-1 drop-shadow-sm">{formatTime(msg.timestamp)}</span>}
                  
                  {/* 吹き出し本体 */}
                  <div className={`max-w-[75vw] px-4 py-2.5 text-[15px] leading-relaxed shadow-sm break-words
                    ${isMine 
                      ? 'bg-[#85e249] text-gray-900 rounded-2xl rounded-tr-sm' // LINE風の緑 
                      : 'bg-white text-gray-900 rounded-2xl rounded-tl-sm'    // 相手は白
                    }`}
                  >
                    {msg.text}
                  </div>
                  
                  {/* 相手のメッセージの時間 */}
                  {!isMine && <span className="text-[10px] text-white/80 mb-1 drop-shadow-sm">{formatTime(msg.timestamp)}</span>}
                </div>
              </div>
            );
          })}
          {/* スクロール用の空要素 */}
          <div ref={messagesEndRef} className="h-1" />
        </div>

        {/* 入力エリア */}
        <form onSubmit={handleSendMessage} className="bg-gray-100 p-2 sm:p-3 flex items-end space-x-2 border-t border-gray-300 pb-safe">
          <textarea 
            value={inputText}
            onChange={e => setInputText(e.target.value)}
            onKeyDown={e => {
              // Enterキーで送信（Shift+Enterで改行）
              if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                handleSendMessage(e);
              }
            }}
            className="flex-1 bg-white rounded-2xl px-4 py-3 focus:outline-none focus:ring-2 focus:ring-green-400 max-h-32 min-h-[44px] resize-none overflow-y-auto text-sm"
            placeholder="メッセージを入力..."
            rows={1}
          />
          <button 
            type="submit" 
            disabled={!inputText.trim()}
            className="bg-[#06c755] text-white p-3 rounded-full disabled:opacity-50 disabled:bg-gray-400 hover:bg-green-600 transition flex-shrink-0 mb-0.5 shadow-sm"
          >
            <Send size={20} className="ml-0.5" />
          </button>
        </form>

      </div>
    </div>
  );
}
