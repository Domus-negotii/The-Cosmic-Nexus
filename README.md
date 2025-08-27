import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInWithCustomToken, signInAnonymously } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp, getDocs, where, deleteDoc, doc } from 'firebase/firestore';

// Main App component
export default function App() {
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [posts, setPosts] = useState([]);
  const [newPostContent, setNewPostContent] = useState('');
  const [userId, setUserId] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // Define cosmic emojis for reactions
  const cosmicEmojis = ['✨', '🌌', '🚀', '👽', '💫'];

  // Initialize Firebase and authenticate user
  useEffect(() => {
    try {
      // These are global variables provided by the Canvas environment.
      const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
      const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

      if (!firebaseConfig) {
        throw new Error('Firebase config is missing. Please ensure it is provided.');
      }

      // Initialize Firebase services
      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const authInstance = getAuth(app);
      setDb(firestore);
      setAuth(authInstance);

      // Authenticate the user.
      const authenticate = async () => {
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(authInstance, initialAuthToken);
          } else {
            await signInAnonymously(authInstance);
          }
          setUserId(authInstance.currentUser.uid);
          setLoading(false);
        } catch (e) {
          setError('Failed to authenticate with Firebase.');
          console.error('Firebase Auth Error:', e);
          setLoading(false);
        }
      };
      authenticate();

    } catch (e) {
      setError('Failed to initialize the app. Check console for details.');
      console.error('Initialization Error:', e);
      setLoading(false);
    }
  }, []);

  // Set up real-time listener for posts
  useEffect(() => {
    if (!db || !userId) return;

    const postsCollectionPath = `artifacts/${typeof __app_id !== 'undefined' ? __app_id : 'default-app-id'}/public/data/posts`;
    const q = query(collection(db, postsCollectionPath), orderBy('timestamp', 'desc'));

    const unsubscribe = onSnapshot(q, (snapshot) => {
      const postsData = snapshot.docs.map(doc => ({
        id: doc.id,
        ...doc.data(),
        // Add a reactions property to each post object to store real-time reaction counts.
        reactions: {},
      }));

      // For each post, set up a real-time listener for its reactions
      postsData.forEach((post) => {
        const reactionsCollectionPath = `${postsCollectionPath}/${post.id}/reactions`;
        onSnapshot(collection(db, reactionsCollectionPath), (reactionSnapshot) => {
          const reactionsCount = {};
          const userReactions = [];
          reactionSnapshot.docs.forEach(reactionDoc => {
            const reactionData = reactionDoc.data();
            reactionsCount[reactionData.emoji] = (reactionsCount[reactionData.emoji] || 0) + 1;
            if (reactionData.userId === userId) {
              userReactions.push(reactionData.emoji);
            }
          });
          
          setPosts(prevPosts =>
            prevPosts.map(prevPost =>
              prevPost.id === post.id
                ? { ...prevPost, reactions: reactionsCount, userReactions: userReactions }
                : prevPost
            )
          );
        });
      });
      setPosts(postsData);
    }, (e) => {
      setError('Failed to load posts in real-time. Check your network connection.');
      console.error('Firestore Real-time Error:', e);
    });

    // Cleanup listener on component unmount
    return () => unsubscribe();
  }, [db, userId]);

  // Handle new post submission
  const handlePostSubmit = async (e) => {
    e.preventDefault();
    if (!newPostContent.trim() || !db || !userId) return;

    try {
      setLoading(true);
      const postsCollectionPath = `artifacts/${typeof __app_id !== 'undefined' ? __app_id : 'default-app-id'}/public/data/posts`;
      await addDoc(collection(db, postsCollectionPath), {
        content: newPostContent,
        timestamp: serverTimestamp(),
        userId: userId,
      });
      setNewPostContent('');
    } catch (e) {
      setError('Failed to post. Check your network or permissions.');
      console.error('Firestore Write Error:', e);
    } finally {
      setLoading(false);
    }
  };

  // Handle reaction submission
  const handleReaction = async (postId, emoji) => {
    if (!db || !userId) return;

    try {
      const reactionsCollectionPath = `artifacts/${typeof __app_id !== 'undefined' ? __app_id : 'default-app-id'}/public/data/posts/${postId}/reactions`;
      const q = query(collection(db, reactionsCollectionPath), where('userId', '==', userId), where('emoji', '==', emoji));
      const existingReaction = await getDocs(q);

      if (!existingReaction.empty) {
        // If the user already reacted, remove their reaction (toggle off)
        const reactionDocRef = doc(db, reactionsCollectionPath, existingReaction.docs[0].id);
        await deleteDoc(reactionDocRef);
      } else {
        // Otherwise, add a new reaction
        await addDoc(collection(db, reactionsCollectionPath), {
          emoji: emoji,
          userId: userId,
        });
      }
    } catch (e) {
      setError('Failed to react. Please try again.');
      console.error('Firestore Reaction Error:', e);
    }
  };


  // UI rendering
  return (
    <div className="bg-slate-950 min-h-screen p-4 sm:p-8 flex flex-col items-center font-inter text-gray-200">
      {/* Main container with max-w for readability */}
      <div className="w-full max-w-2xl bg-slate-900 shadow-xl shadow-indigo-500/20 rounded-2xl p-6 sm:p-8 space-y-8 border border-slate-700">
        {/* Header section */}
        <header className="text-center">
          <h1 className="text-4xl sm:text-5xl font-extrabold text-indigo-400 tracking-tight mb-2">
            The Cosmic Nexus
          </h1>
          <p className="text-lg text-indigo-200">
            A real-time thought network for every sentient being across the cosmos.
          </p>
          <p className="text-sm mt-4 text-slate-400">
            Your unique galactic signature is: <span className="text-xs break-all text-slate-400 font-mono select-all">{userId}</span>
          </p>
        </header>

        {/* Error and loading states */}
        {error && (
          <div className="p-4 bg-red-900 text-red-300 rounded-lg text-center font-semibold border-2 border-red-700">
            {error}
          </div>
        )}
        {loading && (
          <div className="flex items-center justify-center p-4">
            <svg className="animate-spin h-8 w-8 text-indigo-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
              <circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle>
              <path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
            </svg>
          </div>
        )}

        {/* Post creation form */}
        <form onSubmit={handlePostSubmit} className="space-y-4">
          <textarea
            className="w-full p-4 border border-slate-700 rounded-xl bg-slate-800 text-indigo-100 placeholder-indigo-300 focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-all resize-none"
            rows="4"
            placeholder="Broadcast your thought to the universe..."
            value={newPostContent}
            onChange={(e) => setNewPostContent(e.target.value)}
          />
          <button
            type="submit"
            disabled={!newPostContent.trim() || loading}
            className="w-full px-6 py-3 bg-indigo-600 text-white font-semibold rounded-xl shadow-lg shadow-indigo-500/30 hover:bg-indigo-700 transition-colors disabled:bg-slate-700 disabled:text-slate-500 disabled:cursor-not-allowed"
          >
            Transmit Thought
          </button>
        </form>

        {/* Display posts */}
        <main className="space-y-4">
          <h2 className="text-2xl font-bold text-indigo-400 border-b-2 border-slate-700 pb-2">
            Cosmic Streams
          </h2>
          {posts.length === 0 && !loading && !error && (
            <p className="text-center text-slate-500 p-4">
              No cosmic transmissions detected yet. Be the first to broadcast!
            </p>
          )}
          {posts.map(post => (
            <div key={post.id} className="bg-slate-800 p-5 rounded-xl shadow-sm border border-slate-700">
              <p className="text-indigo-100 leading-relaxed whitespace-pre-wrap">{post.content}</p>
              <div className="mt-4 flex flex-wrap gap-2">
                {cosmicEmojis.map(emoji => (
                  <button
                    key={emoji}
                    onClick={() => handleReaction(post.id, emoji)}
                    className={`text-lg p-2 rounded-full transition-all ${
                      post.userReactions && post.userReactions.includes(emoji)
                        ? 'bg-indigo-500 shadow-md'
                        : 'bg-slate-700 hover:bg-slate-600'
                    }`}
                  >
                    {emoji}
                    <span className="ml-1 text-xs text-slate-300">{post.reactions[emoji] || 0}</span>
                  </button>
                ))}
              </div>
              <div className="mt-3 text-xs text-slate-400 flex justify-between items-center">
                <span>
                  Signature: <span className="font-mono">{post.userId}</span>
                </span>
                <span>
                  {post.timestamp ? new Date(post.timestamp.seconds * 1000).toLocaleString() : 'Just now'}
                </span>
              </div>
            </div>
          ))}
        </main>
      </div>
    </div>
  );
}
