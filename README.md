<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>The Cosmic Nexus</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
    import { getAuth, signInWithCustomToken, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
    import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, serverTimestamp, getDocs, where, deleteDoc, doc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

    // Global variables from the canvas environment
    const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : null;
    const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

    let db;
    let auth;
    let userId;

    const cosmicEmojis = ['✨', '🌌', '🚀', '👽', '💫'];
    const postsContainer = document.getElementById('posts-container');
    const newPostForm = document.getElementById('new-post-form');
    const postContentInput = document.getElementById('post-content');
    const signatureDisplay = document.getElementById('signature-display');
    const loadingIndicator = document.getElementById('loading-indicator');
    const errorDisplay = document.getElementById('error-display');

    /**
     * Display an error message to the user.
     * @param {string} message - The error message to display.
     */
    function displayError(message) {
      errorDisplay.textContent = message;
      errorDisplay.classList.remove('hidden');
    }

    /**
     * Hide the error message.
     */
    function hideError() {
      errorDisplay.classList.add('hidden');
      errorDisplay.textContent = '';
    }

    /**
     * Show or hide the loading indicator.
     * @param {boolean} isLoading - True to show, false to hide.
     */
    function setLoading(isLoading) {
      if (isLoading) {
        loadingIndicator.classList.remove('hidden');
      } else {
        loadingIndicator.classList.add('hidden');
      }
    }

    /**
     * Handles the reaction to a post.
     * Toggles a user's reaction on or off.
     * @param {string} postId - The ID of the post.
     * @param {string} emoji - The emoji to react with.
     */
    async function handleReaction(postId, emoji) {
      if (!db || !userId) return;

      try {
        const reactionsCollectionPath = `artifacts/${appId}/public/data/posts/${postId}/reactions`;
        const q = query(collection(db, reactionsCollectionPath), where('userId', '==', userId), where('emoji', '==', emoji));
        const existingReaction = await getDocs(q);

        if (!existingReaction.empty) {
          const reactionDocRef = doc(db, reactionsCollectionPath, existingReaction.docs[0].id);
          await deleteDoc(reactionDocRef);
        } else {
          await addDoc(collection(db, reactionsCollectionPath), {
            emoji: emoji,
            userId: userId,
          });
        }
      } catch (e) {
        displayError('Failed to react. Please try again.');
        console.error('Firestore Reaction Error:', e);
      }
    }

    /**
     * Renders a single post element and its reactions.
     * @param {object} post - The post data.
     * @param {object} reactionsCount - The counts of each emoji reaction.
     * @param {string[]} userReactions - The emojis the current user has reacted with.
     * @returns {HTMLElement} The post element.
     */
    function createPostElement(post, reactionsCount, userReactions) {
      const postDiv = document.createElement('div');
      postDiv.className = "bg-slate-800 p-5 rounded-xl shadow-sm border border-slate-700";

      const postContent = document.createElement('p');
      postContent.className = "text-indigo-100 leading-relaxed whitespace-pre-wrap";
      postContent.textContent = post.content;

      const reactionContainer = document.createElement('div');
      reactionContainer.className = "mt-4 flex flex-wrap gap-2";

      cosmicEmojis.forEach(emoji => {
        const button = document.createElement('button');
        button.className = `text-lg p-2 rounded-full transition-all ${userReactions.includes(emoji) ? 'bg-indigo-500 shadow-md' : 'bg-slate-700 hover:bg-slate-600'}`;
        button.innerHTML = `${emoji}<span class="ml-1 text-xs text-slate-300">${reactionsCount[emoji] || 0}</span>`;
        button.onclick = () => handleReaction(post.id, emoji);
        reactionContainer.appendChild(button);
      });

      const footer = document.createElement('div');
      footer.className = "mt-3 text-xs text-slate-400 flex justify-between items-center";
      footer.innerHTML = `
        <span>Signature: <span class="font-mono">${post.userId}</span></span>
        <span>${post.timestamp ? new Date(post.timestamp.seconds * 1000).toLocaleString() : 'Just now'}</span>
      `;

      postDiv.appendChild(postContent);
      postDiv.appendChild(reactionContainer);
      postDiv.appendChild(footer);

      return postDiv;
    }

    /**
     * Initializes the Firebase app and sets up event listeners.
     */
    window.onload = async function() {
      try {
        if (!firebaseConfig) {
          throw new Error('Firebase config is missing.');
        }

        const app = initializeApp(firebaseConfig);
        db = getFirestore(app);
        auth = getAuth(app);

        // Authenticate the user
        if (initialAuthToken) {
          await signInWithCustomToken(auth, initialAuthToken);
        } else {
          await signInAnonymously(auth);
        }
        userId = auth.currentUser.uid;
        signatureDisplay.textContent = userId;
        setLoading(false);

        // Set up real-time listener for posts
        const postsCollectionPath = `artifacts/${appId}/public/data/posts`;
        const q = query(collection(db, postsCollectionPath), orderBy('timestamp', 'desc'));

        onSnapshot(q, (snapshot) => {
          postsContainer.innerHTML = '';
          if (snapshot.empty) {
            postsContainer.innerHTML = `<p class="text-center text-slate-500 p-4">
              No cosmic transmissions detected yet. Be the first to broadcast!
            </p>`;
            return;
          }
          
          snapshot.docs.forEach(postDoc => {
            const post = { id: postDoc.id, ...postDoc.data() };
            const reactionsCollectionPath = `${postsCollectionPath}/${post.id}/reactions`;
            
            // Set up a real-time listener for reactions of this specific post
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

              // Check if the post element is already in the DOM and update it, otherwise create a new one
              const existingPostElement = document.querySelector(`[data-post-id="${post.id}"]`);
              if (existingPostElement) {
                const newPostElement = createPostElement(post, reactionsCount, userReactions);
                newPostElement.setAttribute('data-post-id', post.id);
                postsContainer.replaceChild(newPostElement, existingPostElement);
              } else {
                const newPostElement = createPostElement(post, reactionsCount, userReactions);
                newPostElement.setAttribute('data-post-id', post.id);
                postsContainer.appendChild(newPostElement);
              }

            }, (e) => {
              console.error('Firestore Reaction Listener Error:', e);
            });
          });
          hideError();
        }, (e) => {
          displayError('Failed to load posts in real-time. Check your network connection.');
          console.error('Firestore Main Listener Error:', e);
        });

        // Handle new post form submission
        newPostForm.addEventListener('submit', async (e) => {
          e.preventDefault();
          const content = postContentInput.value.trim();
          if (!content || !db || !userId) return;

          setLoading(true);
          try {
            const postsCollectionPath = `artifacts/${appId}/public/data/posts`;
            await addDoc(collection(db, postsCollectionPath), {
              content: content,
              timestamp: serverTimestamp(),
              userId: userId,
            });
            postContentInput.value = '';
          } catch (e) {
            displayError('Failed to post. Check your network or permissions.');
            console.error('Firestore Write Error:', e);
          } finally {
            setLoading(false);
          }
        });

      } catch (e) {
        displayError('Failed to initialize the app.');
        console.error('Initialization Error:', e);
      }
    };
  </script>
</head>
<body class="bg-slate-950 min-h-screen p-4 sm:p-8 flex flex-col items-center font-inter text-gray-200">
  <!-- Main container -->
  <div class="w-full max-w-2xl bg-slate-900 shadow-xl shadow-indigo-500/20 rounded-2xl p-6 sm:p-8 space-y-8 border border-slate-700">
    <!-- Header -->
    <header class="text-center">
      <h1 class="text-4xl sm:text-5xl font-extrabold text-indigo-400 tracking-tight mb-2">
        The Cosmic Nexus
      </h1>
      <p class="text-lg text-indigo-200">
        A real-time thought network for every sentient being across the cosmos.
      </p>
      <p class="text-sm mt-4 text-slate-400">
        Your unique galactic signature is: <span id="signature-display" class="text-xs break-all text-slate-400 font-mono select-all">Loading...</span>
      </p>
    </header>

    <!-- Error and loading states -->
    <div id="error-display" class="p-4 bg-red-900 text-red-300 rounded-lg text-center font-semibold border-2 border-red-700 hidden"></div>
    <div id="loading-indicator" class="flex items-center justify-center p-4 hidden">
      <svg class="animate-spin h-8 w-8 text-indigo-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
      </svg>
    </div>

    <!-- Post creation form -->
    <form id="new-post-form" class="space-y-4">
      <textarea
        id="post-content"
        class="w-full p-4 border border-slate-700 rounded-xl bg-slate-800 text-indigo-100 placeholder-indigo-300 focus:outline-none focus:ring-2 focus:ring-indigo-500 transition-all resize-none"
        rows="4"
        placeholder="Broadcast your thought to the universe..."
      ></textarea>
      <button
        type="submit"
        class="w-full px-6 py-3 bg-indigo-600 text-white font-semibold rounded-xl shadow-lg shadow-indigo-500/30 hover:bg-indigo-700 transition-colors"
      >
        Transmit Thought
      </button>
    </form>

    <!-- Display posts -->
    <main class="space-y-4">
      <h2 class="text-2xl font-bold text-indigo-400 border-b-2 border-slate-700 pb-2">
        Cosmic Streams
      </h2>
      <div id="posts-container" class="space-y-4">
        <!-- Posts will be dynamically inserted here by JavaScript -->
      </div>
    </main>
  </div>
</body>
</html>
