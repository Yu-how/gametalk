<!doctype html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>簡約遊戲論壇 (Firebase 同步版)</title>
<style>
:root{--orange:#ff6a00;--bg:#fff;--muted:#6b6b6b;--card-shadow:0 6px 18px rgba(0,0,0,0.06);--radius:14px;font-family:Inter,ui-sans-serif,system-ui,-apple-system,"Segoe UI",Roboto,"Helvetica Neue",Arial;}
*{box-sizing:border-box}
body{margin:0;background:linear-gradient(180deg,#fff 0%, #fffbf6 100%);color:#111}
.container{max-width:1100px;margin:28px auto;padding:20px}
header{display:flex;gap:16px;align-items:center;justify-content:space-between;flex-wrap:wrap}
.brand{display:flex;gap:12px;align-items:center}
.logo{width:52px;height:52px;border-radius:12px;background:var(--orange);display:flex;align-items:center;justify-content:center;color:white;font-weight:700;font-size:22px;box-shadow:var(--card-shadow)}
h1{margin:0;font-size:20px}
p.lead{margin:0;color:var(--muted);font-size:13px}
.layout{display:grid;grid-template-columns:1fr 360px;gap:20px;margin-top:20px}
.card{background:var(--bg);border-radius:var(--radius);padding:18px;box-shadow:var(--card-shadow)}
.composer{display:flex;flex-direction:column;gap:10px}
.composer input[type=text],.composer textarea{width:100%;padding:10px;border-radius:10px;border:1px solid #eee;font-size:14px}
.composer textarea{min-height:110px;resize:vertical}
.row{display:flex;gap:8px;align-items:center}
.btn{background:var(--orange);color:white;padding:10px 14px;border-radius:10px;border:0;cursor:pointer}
.btn.secondary{background:transparent;color:var(--orange);border:1px solid rgba(255,106,0,0.15)}
.posts{display:flex;flex-direction:column;gap:12px;margin-top:14px}
.post{padding:12px;border-radius:12px;border:1px solid #f0f0f0}
.post .meta{display:flex;gap:10px;align-items:center;color:var(--muted);font-size:13px}
.post img.preview{max-width:100%;border-radius:8px;margin-top:10px}
.sidebar h3{margin:0 0 10px 0}
.news-list{display:flex;flex-direction:column;gap:10px}
.news-item{padding:10px;border-radius:10px;border:1px dashed #efe6dc;font-size:14px}
.comments{margin-top:8px;padding-top:8px;border-top:1px dashed #f1f1f1}
.comment{font-size:13px;color:#2b2b2b}
.small{font-size:12px;color:var(--muted)}
.search{display:flex;gap:8px}
.search input{flex:1;padding:10px;border-radius:10px;border:1px solid #eee}
footer{margin-top:28px;color:var(--muted);font-size:13px;text-align:center}
@media(max-width:920px){.layout{grid-template-columns:1fr;}.sidebar{order:2}}
</style>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore-compat.js"></script>
</head>
<body>
<div class="container">
<header>
<div class="brand">
<div class="logo">G</div>
<div>
<h1>簡約遊戲論壇</h1>
<p class="lead">主色：橘 • 簡約風 • 支援貼文、留言與新聞 • Firebase 同步版</p>
</div>
</div>
<div style="min-width:320px; flex:1;">
<div class="search">
<input id="globalSearch" placeholder="搜尋關鍵字（文章 / 留言 / 新聞）..." />
<button class="btn" id="searchBtn">搜尋</button>
</div>
<div id="searchResults" style="margin-top:12px"></div>
</div>
</header>
<div class="layout">
<main>
<section class="card composer">
<input id="postTitle" type="text" placeholder="文章標題" />
<textarea id="postContent" placeholder="寫下你的文章...（支援圖片）"></textarea>
<div class="row">
<input id="postImage" type="file" accept="image/*" />
<select id="postType">
<option value="post">一般文章</option>
<option value="image">圖片文章</option>
</select>
<div style="margin-left:auto">
<button class="btn" id="publishBtn">發佈</button>
<button class="btn secondary" id="clearBtn">清除</button>
</div>
</div>
</section>
<section class="card" style="margin-top:12px">
<h3>文章列表</h3>
<div id="posts" class="posts" aria-live="polite"></div>
</section>
</main>
<aside class="sidebar">
<section class="card">
<h3>即時遊戲新聞</h3>
<p class="small">來源：Google News RSS + NewsAPI 備援</p>
<div style="display:flex;gap:8px;margin-top:8px">
<input id="newsQuery" placeholder="搜尋新聞（ex: Elden Ring）" />
<button class="btn" id="fetchNewsBtn">取得新聞</button>
</div>
<div id="news" class="news-list" style="margin-top:10px"></div>
</section>
</aside>
</div>
<footer>版權 © 簡約遊戲論壇 — Firebase 同步版</footer>
</div>
<script>
// Firebase 初始化
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();
const postsCol = db.collection('posts');

// DOM 元素
const postsEl=document.getElementById('posts');
const publishBtn=document.getElementById('publishBtn');
const clearBtn=document.getElementById('clearBtn');
const postTitle=document.getElementById('postTitle');
const postContent=document.getElementById('postContent');
const postImage=document.getElementById('postImage');
const postType=document.getElementById('postType');
const globalSearch=document.getElementById('globalSearch');
const searchResults=document.getElementById('searchResults');

// 轉 base64
function fileToBase64(file){return new Promise((res,rej)=>{if(!file)return res(null);const reader=new FileReader();reader.onload=()=>res(reader.result);reader.onerror=rej;reader.readAsDataURL(file);});}
function escapeHtml(s){return(s||'').replace(/[&<>"']/g,c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'':'&#39;'}[c]||c));}
function nl2br(s){return(s||'').replace(/\n/g,'<br>');}

// 發布文章
publishBtn.addEventListener('click',async()=>{
  const title=postTitle.value.trim();
  const content=postContent.value.trim();
  if(!title&&!content&&!postImage.files.length){alert('請輸入標題或內文或上傳圖片。');return;}
  const img=postImage.files[0]?await fileToBase64(postImage.files[0]):null;
  await postsCol.add({title,content,img,type:postType.value,createdAt:new Date().toISOString(),comments:[]});
  postTitle.value='';postContent.value='';postImage.value='';
});
clearBtn.addEventListener('click',()=>{postTitle.value='';postContent.value='';postImage.value='';});

// Firestore 監聽
postsCol.orderBy('createdAt','desc').onSnapshot(snapshot=>{
  const posts=snapshot.docs.map(doc=>({id:doc.id,...doc.data()}));
  renderPosts(posts);
});

// 渲染文章
function renderPosts(posts){
  postsEl.innerHTML='';
  if(posts.length===0){postsEl.innerHTML='<div class="small">目前沒有文章。試試發佈第一篇！</div>';return;}
  for(const p of posts){
    const el=document.createElement('div');el.className='post';
    el.innerHTML=`<div style="display:flex;justify-content:space-between;align-items:center">
      <div>
        <div style="font-weight:600">${escapeHtml(p.title||'(無標題)')}</div>
        <div class="meta small">${new Date(p.createdAt).toLocaleString()} • ${p.type==='image'?'圖片文章':'文章'}</div>
      </div>
      <div style="text-align:right"><button class="btn secondary" onclick="deletePost('${p.id}')">刪除</button></div>
    </div>
    <div style="margin-top:8px">${nl2br(escapeHtml(p.content||''))}</div>
    ${p.img? `<img class="preview" src="${p.img}" alt="preview">` : ''}
    <div class="comments">
      <div class="small">留言（${(p.comments||[]).length}）</div>
      <div id="c_${p.id}"></div>
      <div style="display:flex;gap:8px;margin-top:8px">
        <input id="ci_${p.id}" placeholder="你的留言..." />
        <button class="btn" onclick="addComment('${p.id}')">送出</button>
      </div>
    </div>`;
    postsEl.appendChild(el);
    renderComments(p.id,p.comments||[]);
  }
}

function deletePost(id){if(!confirm('確定刪除此文章？')) return; postsCol.doc(id).delete();}
function addComment(postId){
  const input=document.getElementById('ci_'+postId);
  const text=input.value.trim(); if(!text) return;
  const comment={id:'c_'+Date.now(),text,createdAt:new Date().toISOString()};
  postsCol.doc(postId).update({comments:firebase.firestore.FieldValue.arrayUnion(comment)});
  input.value='';
}
function renderComments(postId,comments){
  const container=document.getElementById('c_'+postId); if(!container) return;
  container.innerHTML='';
  for(const c of comments){
    const d=document.createElement('div');d.className='comment';
    d.innerHTML=`<div>${escapeHtml(c.text)}</div><div class="small">${new Date(c.createdAt).toLocaleString()}</div>`;
    container.appendChild(d);
  }
}

// 搜尋功能 (文章 + 留言)
function simpleScore(text,keywords){text=(text||'').toLowerCase();let score=0;for(const k of keywords){if(!k)continue;score+=text.split(k).length-1;}return score;}
function searchAll(q){
  const kws=q.toLowerCase().split(/\s+/).filter(Boolean);
  const results=[];
  postsCol.get().then(snap=>{
    snap.docs.forEach(doc=>{
      const p=doc.data(); const content=[p.title,p.content,(p.comments||[]).map(c=>c.text).join(' ')].join(' ');
      const score=simpleScore(content,kws);
      if(score>0) results.push({type:'post',score,item:{id:doc.id,...p}});
    });
  }).then(()=>showSearchResults(q,results));
}
function showSearchResults(q,results){
  searchResults.innerHTML=`<div style="font-weight:600;margin-bottom:6px">搜尋：${escapeHtml(q)} - 相關文章</div>`;
  if(results.length===0){searchResults.innerHTML+='<div class="small">找不到相關內容</div>';return;}
  results.forEach(r=>{if(r.type==='post'){searchResults.innerHTML+=`<div style="border-bottom:1px solid #eee;padding:4px 0;"><strong>[文章]</strong> ${escapeHtml(r.item.title||'(無標題)')}</div>`;}});
}
document.getElementById('searchBtn').addEventListener('click',()=>{const q=globalSearch.value.trim();if(q) searchAll(q);});
globalSearch.addEventListener('keydown',(e)=>{if(e.key==='Enter') searchBtn.click();});
</script>
</body>
</html>
