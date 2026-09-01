/* ネットワーク通信 (PeerJS) */
    const SESSION_KEY = 'trinity_ttp_session_v1';
    const MAX_RECONNECT_ATTEMPTS = 8;

    // 修正要件：バグ調査の結果、localStorageは同一ブラウザの全タブで共有されるため、
    // 同じブラウザの複数タブで別プレイヤーとして接続すると互いのセッション情報を上書きしてしまい、
    // 「プレイヤー3がプレイヤー2として復帰する」といった取り違えが発生していた。
    // タブごとに独立しているsessionStorageに切り替えることでこの衝突を防ぐ。
    function loadSession() {
      try {
        const raw = sessionStorage.getItem(SESSION_KEY);
        if (!raw) return null;
        const obj = JSON.parse(raw);
        if (!obj || !obj.savedAt) return null;
        if (Date.now() - obj.savedAt > 1000 * 60 * 60 * 6) return null; // 6時間で失効
        return obj;
      } catch (e) { return null; }
    }
    function clearSession() {
      try { sessionStorage.removeItem(SESSION_KEY); } catch (e) {}
    }

    let savedSession = loadSession();

    let myId = '';
    let isHost = false;
    let myPlayerIndex = 0;
    // 修正要件：2人/3人モードの切り替え。playerCountはmatchModeから導出する実プレイヤー人数
    let matchMode = '3p'; // '2p' | '3p'
    let playerCount = 3;
    // 修正要件：再接続時に同じ枠へ戻せるよう、connectionsはプレイヤー番号固定のスロット方式に変更(0は未使用/ホスト自身)
    let connections = [null, null, null];
    let hostConn = null;
    let lastKnownHostId = null;
    let everConnectedOnce = false;
    let reconnectAttempts = 0;
    let reconnectTimer = null;
    let autoSaveTimer = null;
    let currentPhase = 'lobby'; // 'lobby' | 'draft' | 'deckbuilder' | 'battle'

    let playerNames = ['P1', 'P2', 'P3'];
    let myName = '';
    let cardFolder = 'card-bl';

    // 修正要件：ホストが同端末でリロード/クラッシュしても同じIDに復帰できるよう、
    // 保存済みセッションがホストのものであれば同じPeer IDを再取得する
    let peer = (savedSession && savedSession.isHost && savedSession.hostId)
      ? new Peer(savedSession.hostId)
      : new Peer();

    peer.on('open', (id) => {
      myId = id;
      document.getElementById('my-peer-id').innerText = id;
    });

    peer.on('error', (err) => {
      console.error('Peer error', err);
      if (err && err.type === 'unavailable-id') {
        updateLobbyStatus('前回のセッションIDがまだ解放されていません。数秒後にページを再読み込みして「復帰する」をお試しください。');
      } else if (!isHost && lastKnownHostId) {
        scheduleReconnect();
      }
    });

    function copyMyId() {
      if (!myId || myId === '発行中...') return;
      navigator.clipboard.writeText(myId).then(() => {
        const btn = document.getElementById('btn-copy-id');
        btn.innerText = 'コピーしました！';
        btn.style.background = '#10b981';
        btn.style.color = '#fff';
        setTimeout(() => {
          btn.innerText = 'コピー';
          btn.style.background = '#38bdf8';
          btn.style.color = '#0f172a';
        }, 2000);
      });
    }

    function updateLobbyStatus(msg) {
      document.getElementById('lobby-status').innerText = msg;
    }

    /* 修正要件：セッション復帰ボタンの表示 */
    (function setupResumeBox() {
      if (!savedSession) return;
      const box = document.getElementById('resume-session-box');
      if (box) box.style.display = 'block';
    })();

    /* 修正要件：設定できる名前の文字数を全角6文字、半角12文字に制限 */
    const MAX_NAME_WEIGHT = 12; // 全角(2)×6文字 or 半角(1)×12文字 相当
    function getNameWeight(str) {
      let weight = 0;
      for (const ch of str) {
        const code = ch.codePointAt(0);
        // 半角英数記号・半角カタカナは1、それ以外(全角)は2としてカウント
        const isHalfWidth = (code >= 0x0020 && code <= 0x007E) || (code >= 0xFF61 && code <= 0xFF9F);
        weight += isHalfWidth ? 1 : 2;
      }
      return weight;
    }
    function isNameLengthValid(str) {
      return getNameWeight(str) <= MAX_NAME_WEIGHT;
    }
    (function setupNameInputLimit() {
      const el = document.getElementById('player-name-input');
      if (!el) return;
      el.addEventListener('input', () => {
        let val = el.value;
        while (getNameWeight(val) > MAX_NAME_WEIGHT) {
          val = val.slice(0, -1);
        }
        el.value = val;
      });
    })();

    // 修正要件：2人/3人モードのチェックボックスは排他的に切り替える
    function toggleMatchMode(mode) {
      const cb2 = document.getElementById('mode-2p-checkbox');
      const cb3 = document.getElementById('mode-3p-checkbox');
      if (mode === '2p') {
        cb2.checked = true;
        cb3.checked = false;
        matchMode = '2p';
      } else {
        cb3.checked = true;
        cb2.checked = false;
        matchMode = '3p';
      }
    }

    function createRoom() {
      const nameInput = document.getElementById('player-name-input').value.trim();
      if (!nameInput) return alert('プレイヤー名を入力してください');
      if (!isNameLengthValid(nameInput)) return alert('プレイヤー名は全角6文字（半角12文字）以内で入力してください');
      myName = nameInput;
      playerNames[0] = myName;

      isHost = true;
      myPlayerIndex = 0;
      // 修正要件：作成時に選んだモードを反映（人数はここで確定させる）
      playerCount = (matchMode === '2p') ? 2 : 3;
      updateLobbyStatus(`部屋を作成しました (${myName})。他のプレイヤーの接続を待っています... (1/${playerCount})`);
      startAutoSaveLoop();
    }

    function joinRoom() {
      const nameInput = document.getElementById('player-name-input').value.trim();
      if (!nameInput) return alert('プレイヤー名を入力してください');
      if (!isNameLengthValid(nameInput)) return alert('プレイヤー名は全角6文字（半角12文字）以内で入力してください');
      myName = nameInput;

      const hostId = document.getElementById('host-id-input').value.trim();
      if (!hostId) return alert('ホストIDを入力してください');
      isHost = false;
      lastKnownHostId = hostId;
      updateLobbyStatus('ホストに接続中...');
      hostConn = peer.connect(hostId);
      attachHostConnHandlers();
      startAutoSaveLoop();
    }

    /* 修正要件：セッション復帰 */
    function resumeSession() {
      if (!savedSession) return;
      const s = savedSession;

      if (s.phase === 'draft' || s.phase === 'deckbuilder') {
        updateLobbyStatus('ドラフト／デッキ構築中のセッションの復帰には対応していません。お手数ですが最初からやり直してください。');
        clearSession();
        savedSession = null;
        const box = document.getElementById('resume-session-box');
        if (box) box.style.display = 'none';
        return;
      }

      myName = s.myName;
      isHost = s.isHost;
      myPlayerIndex = s.myPlayerIndex;
      cardFolder = s.cardFolder || 'card-bl';
      if (s.playerNames) playerNames = s.playerNames;
      lastKnownHostId = s.hostId;

      const box = document.getElementById('resume-session-box');
      if (box) box.style.display = 'none';

      if (s.private) applyPrivateSnapshot(s.private);

      if (isHost) {
        playerNames[0] = myName;
        updateLobbyStatus('セッションを復帰しています。他のプレイヤーの再接続を待っています...');
        if (s.phase === 'battle' && s.public) {
          applyPublicSnapshot(s.public);
          enterBattleViewOnly();
        }
      } else {
        updateLobbyStatus('ホストに再接続しています...');
        if (s.phase === 'battle') {
          enterBattleViewOnly();
        }
        hostConn = peer.connect(s.hostId);
        attachHostConnHandlers();
      }
      startAutoSaveLoop();
    }

    function enterBattleViewOnly() {
      document.getElementById('lobby-view').style.display = 'none';
      document.getElementById('draft-view').style.display = 'none';
      document.getElementById('deck-builder-view').style.display = 'none';
      document.getElementById('battle-view').style.display = 'flex';
      document.getElementById('header-bar').style.display = 'none';
      document.getElementById('battle-view').classList.toggle('two-player-mode', playerCount === 2);
      currentPhase = 'battle';
      createAllLockSlots();
      adjustPerspectiveBarLayout();
      renderScoreBar();
      updateDeckStatus();
      updateLifeStatus();
    }

    /* 修正要件：ホスト側 - 新規接続の受け付け（枠が空いていれば再接続として認識） */
    peer.on('connection', (conn) => {
      if (!isHost) return;
      conn.on('data', (data) => handleNetworkMessage(data, conn));
      conn.on('close', () => handleHostSideConnLost(conn));
      conn.on('error', () => handleHostSideConnLost(conn));
    });

    function handleHostSideConnLost(conn) {
      const idx = connections.indexOf(conn);
      if (idx !== -1) {
        connections[idx] = null;
        updateConnStatusBanner();
        broadcastLog(`${playerNames[idx] || 'プレイヤー' + idx}との接続が切れました。再接続を待っています…`);
      }
    }

    /* 修正要件：クライアント側 - ホストとの接続確立/切断/自動再接続 */
    function attachHostConnHandlers() {
      hostConn.on('open', () => {
        clearTimeout(reconnectTimer);
        reconnectAttempts = 0;
        lastKnownHostId = hostConn.peer;
        if (everConnectedOnce) {
          setConnBanner('ホストに再接続しました', 'ok');
          setTimeout(hideConnBanner, 2500);
        } else {
          hideConnBanner();
        }
        everConnectedOnce = true;
        hostConn.send({ type: 'HELLO', payload: { name: myName, resumeIndex: getSavedResumeIndex() } });
      });
      hostConn.on('data', (data) => handleNetworkMessage(data, hostConn));
      hostConn.on('close', () => scheduleReconnect());
      hostConn.on('error', () => scheduleReconnect());
    }

    function getSavedResumeIndex() {
      try {
        const raw = sessionStorage.getItem(SESSION_KEY);
        if (!raw) return null;
        const s = JSON.parse(raw);
        if (s && !s.isHost && s.hostId === (hostConn && hostConn.peer) && typeof s.myPlayerIndex === 'number') {
          return s.myPlayerIndex;
        }
      } catch (e) {}
      return null;
    }

    function scheduleReconnect() {
      if (isHost || !lastKnownHostId) return;
      if (reconnectAttempts >= MAX_RECONNECT_ATTEMPTS) {
        setConnBanner('ホストに接続できませんでした。ページを再読み込みしてやり直してください。', 'bad');
        return;
      }
      reconnectAttempts++;
      setConnBanner(`ホストとの接続が切れました。再接続を試みています…(${reconnectAttempts}/${MAX_RECONNECT_ATTEMPTS})`, 'bad');
      clearTimeout(reconnectTimer);
      const delay = Math.min(1500 * reconnectAttempts, 8000);
      reconnectTimer = setTimeout(() => {
        if (isHost || !peer || peer.destroyed) return;
        hostConn = peer.connect(lastKnownHostId);
        attachHostConnHandlers();
      }, delay);
    }

    function setConnBanner(msg, kind) {
      const el = document.getElementById('conn-status-banner');
      if (!el) return;
      el.innerText = msg;
      el.classList.toggle('ok', kind === 'ok');
      el.style.display = 'block';
    }
    function hideConnBanner() {
      const el = document.getElementById('conn-status-banner');
      if (el) el.style.display = 'none';
    }
    function updateConnStatusBanner() {
      if (!isHost) return;
      const missing = [1, 2].filter(i => !connections[i] || !connections[i].open);
      if (missing.length === 0) {
        hideConnBanner();
      } else {
        const names = missing.map(i => playerNames[i] || `プレイヤー${i + 1}`).join('、');
        setConnBanner(`${names} との接続が切れています。再接続を待っています…`, 'bad');
      }
    }

    function broadcast(data, senderConn = null) {
      if (isHost) {
        if (!senderConn) {
          handleNetworkMessage(data, null);
        }
        connections.forEach(c => {
          if (c && c.open && c !== senderConn) {
            c.send(data);
          }
        });
      } else {
        handleNetworkMessage(data, null);
        if (hostConn && hostConn.open) hostConn.send(data);
      }
    }

    /* 修正要件：セッションの保存/復元（ホストの同端末リロード復帰・クライアント再接続用） */
    function buildPrivateSnapshot() {
      const handCards = Array.from(document.querySelectorAll('#hand-cards .dummy-card'))
        .map(el => el.__cardRef).filter(Boolean);
      return {
        deckCards, lifeDecks, localManaList,
        myGraveyard: graveyards[myPlayerIndex],
        myPlayerState: playerStates[myPlayerIndex],
        handCards
      };
    }

    function applyPrivateSnapshot(p) {
      if (!p) return;
      deckCards = p.deckCards || [];
      lifeDecks = p.lifeDecks || { left: [], right: [] };
      localManaList = p.localManaList || [];
      if (p.myGraveyard) graveyards[myPlayerIndex] = p.myGraveyard;
      if (p.myPlayerState) playerStates[myPlayerIndex] = Object.assign({}, playerStates[myPlayerIndex], p.myPlayerState);

      const handContainer = document.getElementById('hand-cards');
      if (handContainer) handContainer.innerHTML = '';
      (p.handCards || []).forEach(card => addCardToHand(card));

      renderManaZone();
      updateDeckStatus();
      updateLifeStatus();
    }

    function buildBoardCardsSnapshot() {
      const table = document.getElementById('zone-table');
      if (!table) return [];
      return Array.from(table.querySelectorAll('.placed-card')).map(el => ({
        id: el.id,
        card: el.cardDataObj,
        ownerIndex: parseInt(el.getAttribute('data-owner')),
        faceDown: el.getAttribute('data-facedown') === 'true',
        tapped: el.getAttribute('data-tapped') === 'true',
        slotOwnerIndex: el.getAttribute('data-slot-owner') !== '' ? parseInt(el.getAttribute('data-slot-owner')) : null,
        slotIndex: el.getAttribute('data-slot-idx') !== '' ? parseInt(el.getAttribute('data-slot-idx')) : null,
        x: parseFloat(el.style.left) || 0,
        y: parseFloat(el.style.top) || 0,
        zIndex: parseInt(el.style.zIndex) || 10
      }));
    }

    function buildPublicSnapshot() {
      return {
        playerNames, cardFolder,
        playerStates, graveyards,
        boardCards: buildBoardCardsSnapshot(),
        pairStage: Object.assign({}, pairStage),
        turnPlayerIndex: (typeof turnPlayerIndex !== 'undefined') ? turnPlayerIndex : null
      };
    }

    function applyPublicSnapshot(pub) {
      if (!pub) return;
      if (pub.playerNames) playerNames = pub.playerNames;
      if (pub.cardFolder) cardFolder = pub.cardFolder;
      if (pub.playerStates) playerStates = pub.playerStates;
      if (pub.graveyards) graveyards = pub.graveyards;
      document.querySelectorAll('.placed-card').forEach(el => el.remove());
      (pub.boardCards || []).forEach(data => syncBoardCardLocal(data));
      if (pub.pairStage) {
        pairStage = pub.pairStage;
        renderAllBarsForMe();
      }
      // 修正要件：再接続/復帰時にもターン状態を復元（アナウンスは出さない）
      if (pub.turnPlayerIndex !== null && pub.turnPlayerIndex !== undefined && typeof applyTurnPlayerSilent === 'function') {
        applyTurnPlayerSilent(pub.turnPlayerIndex);
      }
      updateAllPlayerStatsDisplay();
      renderGraveyardList();
    }

    function saveSession() {
      if (currentPhase === 'lobby' && !myName) return;
      if (isHost && !myId) return; // Peer IDがまだ発行されていない場合は保存しない
      try {
        const session = {
          savedAt: Date.now(),
          isHost, myPlayerIndex, myName,
          hostId: isHost ? myId : lastKnownHostId,
          cardFolder, playerNames,
          phase: currentPhase,
          private: (currentPhase === 'battle') ? buildPrivateSnapshot() : null,
          public: (isHost && currentPhase === 'battle') ? buildPublicSnapshot() : null
        };
        sessionStorage.setItem(SESSION_KEY, JSON.stringify(session));
      } catch (e) { /* 保存できなくても致命的ではないため無視 */ }
    }

    function startAutoSaveLoop() {
      clearInterval(autoSaveTimer);
      autoSaveTimer = setInterval(saveSession, 5000);
      window.addEventListener('beforeunload', saveSession);
    }

    /* ドラフト関連データ */
    let currentRound = 1;
    let currentPick = 1;
    let playerPacks = [[], [], []];
    let selectedCard = null;
    let pickedCards = [];
    let playerPickReady = [false, false, false];
    let playerSelectedCards = [null, null, null];
    let playerDeckReady = [false, false, false];

    function openModeSelectModal() {
      if (isHost) {
        document.getElementById('mode-select-modal').style.display = 'flex';
      }
    }

    function selectCardMode(mode) {
      cardFolder = (mode === 'phantom') ? 'card-ph' : 'card-bl';
      document.getElementById('mode-select-modal').style.display = 'none';
      
      broadcast({
        type: 'SET_CARD_MODE',
        payload: { folder: cardFolder }
      });

      startDraft();
    }

    function buildMasterDeck() {
      masterDeck = [];
      for (let i = 1; i <= 351; i++) {
        masterDeck.push({
          id: `c_${i}`,
          img: `${cardFolder}/c_${i}.jpg`
        });
      }

      SPECIAL_CARDS = [
        { id: 'sp_1', img: `${cardFolder}/sp_1.jpg` },
        { id: 'sp_2', img: `${cardFolder}/sp_2.jpg` },
        { id: 'sp_3', img: `${cardFolder}/sp_3.jpg` },
        { id: 'sp_4', img: `${cardFolder}/sp_4.jpg` }
      ];
    }

    function startDraft() {
      document.getElementById('lobby-view').style.display = 'none';
      buildMasterDeck();

      let deck = [...masterDeck];
      shuffle(deck);

      // 修正要件：2人モードは6パック(2人×3回)、3人モードは9パック(3人×3回)
      const totalPacks = playerCount * 3;
      let selectedPool = deck.slice(0, totalPacks * 13);
      let allPacks = [];
      for (let i = 0; i < totalPacks; i++) {
        allPacks.push(selectedPool.splice(0, 13));
      }

      const draftInitData = {
        type: 'START_DRAFT',
        payload: { allPacks: allPacks, playerNames: playerNames }
      };

      if (isHost) {
        initDraftLocal(allPacks, playerNames);
        connections.forEach((c) => {
          if (c && c.open) {
            c.send(draftInitData);
          }
        });
      }
    }

    let roundPacksPool = [];

    function initDraftLocal(allPacks, names) {
      document.getElementById('lobby-view').style.display = 'none';
      currentPhase = 'draft';
      if (names) playerNames = names;
      // 修正要件：2人モードで3枠目の未読み込みにより永久に「全員ピック完了」判定が
      // 成立しなくなるバグを防ぐため、プレイヤー人数ぶんに配列サイズを揃える
      playerPickReady = new Array(playerCount).fill(false);
      playerSelectedCards = new Array(playerCount).fill(null);
      roundPacksPool = allPacks;
      startRound(1);
    }

    function startRound(roundNum) {
      currentRound = roundNum;
      currentPick = 1;

      // 修正要件：1ラウンドあたりに配るパック数をplayerCountに合わせる
      const baseIdx = (roundNum - 1) * playerCount;
      playerPacks = [];
      for (let i = 0; i < playerCount; i++) {
        playerPacks.push(roundPacksPool[baseIdx + i]);
      }

      updateDraftUI();
      renderPackCards();
    }

    function updateDraftUI() {
      document.getElementById('round-num').innerText = currentRound;
      document.getElementById('pick-num').innerText = currentPick;
      // 修正要件：残パック数の計算をplayerCountに合わせる（2人=6パック/3人=9パック）
      const totalPacks = playerCount * 3;
      document.getElementById('remaining-packs').innerText = totalPacks - (currentRound - 1) * playerCount - Math.floor((currentPick - 1) / 13) * playerCount;
      document.getElementById('pack-count-label').innerText = `残 ${playerPacks[myPlayerIndex].length} 枚`;

      // 修正要件：2人モードではプレイヤー3の決定状況行を非表示にする
      const p3Row = document.getElementById('p3-status-row');
      if (p3Row) p3Row.style.display = (playerCount === 3) ? 'flex' : 'none';

      for (let i = 0; i < playerCount; i++) {
        const nameEl = document.getElementById(`p${i + 1}-name-label`);
        if (nameEl) nameEl.innerText = playerNames[i] || `P${i + 1}`;

        const statusEl = document.getElementById(`p${i + 1}-status`);
        if (statusEl) {
          if (playerPickReady[i]) {
            statusEl.className = 'ready-tag tag-ready';
            statusEl.innerText = '決定済み';
          } else {
            statusEl.className = 'ready-tag tag-waiting';
            statusEl.innerText = '選択中...';
          }
        }
      }
    }

    function renderPackCards() {
      const container = document.getElementById('pack-cards');
      container.innerHTML = '';
      selectedCard = null;
      document.getElementById('confirm-btn').disabled = true;
      document.getElementById('confirm-btn').innerText = 'カードを選択してください';

      const currentPack = playerPacks[myPlayerIndex] || [];
      currentPack.forEach(card => {
        const cardEl = createCardElement(card, () => selectPackCard(card, cardEl));
        container.appendChild(cardEl);
      });
    }

    function selectPackCard(card, el) {
      if (playerPickReady[myPlayerIndex]) return;
      document.querySelectorAll('#pack-cards .card-item').forEach(c => c.classList.remove('selected'));
      el.classList.add('selected');
      selectedCard = card;

      const confirmBtn = document.getElementById('confirm-btn');
      confirmBtn.disabled = false;
      confirmBtn.innerText = 'このカードに決定';
    }

    function confirmPick() {
      if (!selectedCard || playerPickReady[myPlayerIndex]) return;
      
      playerPickReady[myPlayerIndex] = true;
      playerSelectedCards[myPlayerIndex] = selectedCard;
      document.getElementById('confirm-btn').disabled = true;
      document.getElementById('confirm-btn').innerText = '他のプレイヤーを待機中...';

      broadcast({
        type: 'PLAYER_PICK_READY',
        payload: { playerIndex: myPlayerIndex, selectedCard: selectedCard }
      });
    }

    function processPassPacks() {
      for (let i = 0; i < playerCount; i++) {
        const picked = playerSelectedCards[i];
        if (picked) {
          if (i === myPlayerIndex) {
            pickedCards.push(picked);
          }
          const p = playerPacks[i];
          const idx = p.findIndex(c => c.id === picked.id);
          if (idx !== -1) p.splice(idx, 1);
        }
      }

      renderPickedCards();

      // 修正要件：プレイヤー数に応じてパックを1人分ずつ回す（各プレイヤーは1つ前の人からパックを受け取る）
      const newPacks = [];
      for (let i = 0; i < playerCount; i++) {
        newPacks.push(playerPacks[(i - 1 + playerCount) % playerCount]);
      }
      playerPacks = newPacks;

      playerPickReady = new Array(playerCount).fill(false);
      playerSelectedCards = new Array(playerCount).fill(null);

      currentPick++;
      if (currentPick > 13) {
        if (currentRound < 3) {
          startRound(currentRound + 1);
        } else {
          initDeckBuilder();
        }
      } else {
        updateDraftUI();
        renderPackCards();
      }
    }

    function renderPickedCards() {
      const container = document.getElementById('picked-cards');
      container.innerHTML = '';
      document.getElementById('picked-count').innerText = pickedCards.length;

      pickedCards.forEach(card => {
        const el = createCardElement(card);
        container.appendChild(el);
      });
    }

    /* デッキ構築フェーズ */
    function initDeckBuilder() {
      currentPhase = 'deckbuilder';
      document.getElementById('draft-view').style.display = 'none';
      document.getElementById('deck-builder-view').style.display = 'flex';
      document.getElementById('phase-title').innerText = '🛠 デッキ構築フェーズ';
      document.getElementById('draft-status-info').style.display = 'none';
      document.getElementById('deck-status-info').style.display = 'flex';

      // 修正要件：デッキ準備完了判定も2人モードで正しく成立するよう配列サイズを揃える
      playerDeckReady = new Array(playerCount).fill(false);

      poolCards = [...pickedCards];
      mainDeck = [];

      renderDeckBuilder();
      renderSpecialCards();
    }

    function renderDeckBuilder() {
      const poolContainer = document.getElementById('pool-cards');
      poolContainer.innerHTML = '';
      poolCards.forEach(card => {
        const cardEl = createCardElement(card, () => moveToDeck(card));
        poolContainer.appendChild(cardEl);
      });

      const mainContainer = document.getElementById('main-deck-cards');
      mainContainer.innerHTML = '';
      mainDeck.forEach(card => {
        const cardEl = createCardElement(card, () => moveToPool(card));
        mainContainer.appendChild(cardEl);
      });

      document.getElementById('pool-count').innerText = `${poolCards.length}枚`;
      document.getElementById('main-deck-num').innerText = mainDeck.length;
      document.getElementById('main-count').innerText = mainDeck.length;
      document.getElementById('life-count').innerText = poolCards.length;

      validateDeck();
    }

    function renderSpecialCards() {
      const container = document.getElementById('special-card-list');
      container.innerHTML = '';
      SPECIAL_CARDS.forEach(sp => {
        const item = document.createElement('div');
        item.className = 'special-card-item' + (selectedSpecialCard === sp ? ' active' : '');
        item.innerHTML = `<img src="${sp.img}" style="width:100%; height:100%;" alt="sp">`;
        
        item.onclick = () => {
          selectedSpecialCard = sp;
          renderSpecialCards();
          validateDeck();
        };

        item.onmouseenter = (e) => showPreview(sp, e);
        item.onmousemove = (e) => movePreview(e);
        item.onmouseleave = () => hidePreview();

        container.appendChild(item);
      });
    }

    function moveToDeck(card) {
      if (mainDeck.length >= 23) return;
      const idx = poolCards.findIndex(c => c.id === card.id);
      if (idx !== -1) {
        poolCards.splice(idx, 1);
        mainDeck.push(card);
        renderDeckBuilder();
      }
    }

    function moveToPool(card) {
      const idx = mainDeck.findIndex(c => c.id === card.id);
      if (idx !== -1) {
        mainDeck.splice(idx, 1);
        poolCards.push(card);
        renderDeckBuilder();
      }
    }

    function allowDrop(ev) { ev.preventDefault(); }
    function leaveDrop(ev) {}
    function handleDrop(ev, targetZone) {
      ev.preventDefault();
      const cardDataStr = ev.dataTransfer.getData('text/plain');
      if (!cardDataStr) return;
      try {
        const data = JSON.parse(cardDataStr);
        if (data.card) {
          if (targetZone === 'main') moveToDeck(data.card);
          else if (targetZone === 'pool') moveToPool(data.card);
        } else {
          if (targetZone === 'main') moveToDeck(data);
          else if (targetZone === 'pool') moveToPool(data);
        }
      } catch(e) {}
    }

    function validateDeck() {
      const btn = document.getElementById('start-game-btn');
      if (mainDeck.length === 23 && selectedSpecialCard !== null) {
        btn.disabled = false;
        btn.innerText = '対戦準備完了';
      } else {
        btn.disabled = true;
        btn.innerText = `準備中 (残り: ${23 - mainDeck.length}枚 / 固定枠未選択)`;
      }
    }

    function confirmDeckReady() {
      // 修正要件：確定後は編集できない取り消せない操作のため実行前に確認する
      if (!confirm('デッキを確定します。確定後は編集できません。よろしいですか？')) return;
      playerDeckReady[myPlayerIndex] = true;
      document.getElementById('start-game-btn').disabled = true;
      document.getElementById('start-game-btn').innerText = '他のプレイヤーの準備待機中...';

      broadcast({
        type: 'PLAYER_DECK_READY',
        payload: { playerIndex: myPlayerIndex }
      });

      checkAllDeckReady();
    }

    let gameHasStarted = false;
    function checkAllDeckReady() {
      if (gameHasStarted) return;
      if (playerDeckReady.every(r => r === true)) {
        gameHasStarted = true;
        adjustPerspectiveBarLayout();
        renderScoreBar();
        startGame();
        // 修正要件：対戦開始時に先行プレイヤーを決定し、棒の位置を自動調整する（ホストのみが決定し全員に配信）
        // 3人モード：先行は2番手・3番手それぞれに微不利、2番手は3番手に微不利
        // 2人モード：先行の視点で左の8段階棒は下から4段階目、右の4段階棒は下から2段階目に自動移動
        // (段階1=有利/+2, 2=微有利/+1, 3=微不利/-1, 4=不利/-2 の対応。8段階棒は1=最も相手に有利,8=最も自分に有利)
        if (isHost) {
          const first = Math.floor(Math.random() * playerCount);
          if (playerCount === 2) {
            const second = (first + 1) % 2;
            setPairFavor(first, second, 3);           // 右の棒：下から2段階目(=stage3)
            setSecondaryPairFavor(first, second, 5);   // 左の棒：下から4段階目(=stage5、8段階中)
            broadcast({ type: 'SYNC_PAIR_STAGE_ALL', payload: { pairStage: Object.assign({}, pairStage) } });
            broadcast({ type: 'SYNC_SECONDARY_BAR', payload: { stage: secondaryPairStage } });
            renderAllBarsForMe();
            renderSecondaryBarForMe();

            // 修正要件：2人対戦はターン開始前にマリガンを行う。先行プレイヤーのターン開始は
            // 両者のマリガンが完了してから(MULLIGAN_CONFIRMEDが揃ってから)行う
            pendingFirstPlayer = first;
            mulliganConfirmedCount = 0;
            broadcast({ type: 'START_MULLIGAN', payload: {} });
          } else {
            const second = (first + 1) % 3;
            const third = (first + 2) % 3;
            broadcast({ type: 'SET_TURN_PLAYER', payload: { index: first } });
            setPairFavor(first, second, 3);
            setPairFavor(first, third, 3);
            setPairFavor(second, third, 3);
            broadcast({ type: 'SYNC_PAIR_STAGE_ALL', payload: { pairStage: Object.assign({}, pairStage) } });
            renderAllBarsForMe();
          }
        }
      }
    }

    // 修正要件：2人対戦専用のマリガン処理
    let pendingFirstPlayer = null;
    let mulliganConfirmedCount = 0;
    let mulliganSelectedIds = new Set();

    function showMulliganModal() {
      const modal = document.getElementById('mulligan-modal');
      const container = document.getElementById('mulligan-cards');
      if (!modal || !container) return;
      container.innerHTML = '';
      mulliganSelectedIds = new Set();

      // 修正要件：固定カードを除いた初期手札5枚だけを対象にする
      const wrappers = Array.from(document.querySelectorAll('#hand-cards .hand-card-wrapper'))
        .filter(w => w.dataset.fixed !== 'true');

      wrappers.forEach(wrapper => {
        const cardEl = wrapper.querySelector('.dummy-card');
        const card = cardEl ? cardEl.__cardRef : null;
        if (!card) return;
        const el = document.createElement('div');
        el.className = 'mulligan-card';
        el.innerHTML = `<img src="${card.img}" alt="card">`;
        el.onclick = () => {
          if (mulliganSelectedIds.has(card.id)) {
            mulliganSelectedIds.delete(card.id);
            el.classList.remove('selected');
          } else {
            if (mulliganSelectedIds.size >= 5) return;
            mulliganSelectedIds.add(card.id);
            el.classList.add('selected');
          }
          updateMulliganConfirmLabel();
        };
        container.appendChild(el);
      });

      updateMulliganConfirmLabel();
      modal.style.display = 'flex';
    }

    function updateMulliganConfirmLabel() {
      const btn = document.getElementById('mulligan-confirm-btn');
      if (btn) btn.innerText = `決定（${mulliganSelectedIds.size}枚をデッキに戻す）`;
    }

    function confirmMulligan() {
      const handContainer = document.getElementById('hand-cards');
      const count = mulliganSelectedIds.size;

      // 修正要件：選択したカードをデッキに戻してシャッフルし、同じ枚数をデッキの上から引き直す
      Array.from(handContainer.querySelectorAll('.hand-card-wrapper')).forEach(wrapper => {
        const cardEl = wrapper.querySelector('.dummy-card');
        const card = cardEl ? cardEl.__cardRef : null;
        if (card && mulliganSelectedIds.has(card.id)) {
          deckCards.push(card);
          wrapper.remove();
        }
      });
      shuffle(deckCards);

      for (let i = 0; i < count; i++) {
        if (deckCards.length > 0) addCardToHand(deckCards.pop());
      }
      updateDeckStatus();
      broadcastPlayerState();
      broadcastLog(`${myDisplayName()}がマリガンで${count}枚を引き直しました`);

      const modal = document.getElementById('mulligan-modal');
      if (modal) modal.style.display = 'none';

      broadcast({ type: 'MULLIGAN_CONFIRMED', payload: {} });
    }

    let playerStates = [
      { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 },
      { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 },
      { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 }
    ];

    function handleNetworkMessage(data, senderConn) {
      if (isHost && senderConn) {
        connections.forEach(c => {
          if (c && c.open && c !== senderConn) {
            c.send(data);
          }
        });
      }

      switch (data.type) {
        case 'INIT_PLAYER':
          myPlayerIndex = data.payload.index;
          // 修正要件：ホストが決めたモード(人数)をクライアント側にも反映する
          if (data.payload.playerCount) {
            playerCount = data.payload.playerCount;
            matchMode = (playerCount === 2) ? '2p' : '3p';
          }
          break;
        case 'HELLO': {
          // 修正要件：新規参加/再接続の受付。resumeIndexが空いていればその枠に復帰させる
          if (!isHost || !senderConn) break;
          let assignIndex = null;
          const resumeIndex = data.payload.resumeIndex;
          const maxIndex = playerCount - 1; // 修正要件：2人モードでは枠は1つだけ(index=1)
          if (resumeIndex !== null && resumeIndex >= 1 && resumeIndex <= maxIndex && (!connections[resumeIndex] || !connections[resumeIndex].open)) {
            assignIndex = resumeIndex;
          }
          if (assignIndex === null) {
            for (let i = 1; i <= maxIndex; i++) {
              if (!connections[i] || !connections[i].open) { assignIndex = i; break; }
            }
          }
          if (assignIndex === null) {
            senderConn.send({ type: 'ROOM_FULL' });
            setTimeout(() => senderConn.close(), 300);
            break;
          }
          connections[assignIndex] = senderConn;
          playerNames[assignIndex] = data.payload.name || playerNames[assignIndex] || `P${assignIndex + 1}`;
          senderConn.send({ type: 'INIT_PLAYER', payload: { index: assignIndex, playerCount: playerCount } });
          updateConnStatusBanner();
          updateLobbyStatus(`プレイヤーが参加しました (${playerNames.filter(n => n).length}/${playerCount})\n${playerNames.slice(0, playerCount).join(', ')}`);
          broadcastLog(`${playerNames[assignIndex]}が接続しました`);

          if (currentPhase !== 'lobby') {
            // ゲーム開始後の(再)接続 → 公開状態をまとめて送って追いつかせる
            senderConn.send({ type: 'RESYNC_STATE', payload: buildPublicSnapshot() });
          } else {
            let allConnected = true;
            for (let i = 1; i <= maxIndex; i++) {
              if (!connections[i] || !connections[i].open) { allConnected = false; break; }
            }
            if (allConnected && playerNames.slice(0, playerCount).filter(n => n).length === playerCount) {
              updateLobbyStatus('全員揃いました！カードモードを選択してください...');
              openModeSelectModal();
            }
          }
          break;
        }
        case 'ROOM_FULL':
          updateLobbyStatus('この部屋はすでに満員です。');
          break;
        case 'RESYNC_STATE':
          // 修正要件：再接続時にホストから届く公開状態(場のカード・墓地・スコア等)を反映
          applyPublicSnapshot(data.payload);
          if (currentPhase !== 'battle') {
            enterBattleViewOnly();
          }
          break;
        case 'SET_CARD_MODE':
          cardFolder = data.payload.folder;
          buildMasterDeck();
          break;
        case 'START_DRAFT':
          if (data.payload.playerNames) playerNames = data.payload.playerNames;
          initDraftLocal(data.payload.allPacks, data.payload.playerNames);
          break;
        case 'PLAYER_PICK_READY':
          playerPickReady[data.payload.playerIndex] = true;
          playerSelectedCards[data.payload.playerIndex] = data.payload.selectedCard;
          updateDraftUI();
          if (playerPickReady.every(r => r === true)) {
            const notice = document.getElementById('pass-notice');
            notice.style.display = 'block';
            setTimeout(() => {
              notice.style.display = 'none';
              processPassPacks();
            }, 1000);
          }
          break;
        case 'PLAYER_DECK_READY':
          playerDeckReady[data.payload.playerIndex] = true;
          checkAllDeckReady();
          break;
        case 'SYNC_BOARD_CARD':
          syncBoardCardLocal(data.payload);
          break;
        case 'MOVE_BOARD_CARD':
          moveBoardCardLocal(data.payload);
          break;
        case 'CHANGE_CARD_OWNER':
          changeCardOwnerLocal(data.payload);
          break;
        case 'REMOVE_BOARD_CARD':
          removeBoardCardLocal(data.payload.id);
          break;
        case 'SYNC_PLAYER_STATE':
          playerStates[data.payload.index] = data.payload.state;
          updateAllPlayerStatsDisplay();
          break;
        case 'SYNC_GRAVEYARD':
          graveyards[data.payload.ownerIndex] = data.payload.graveyard;
          renderGraveyardList();
          break;
        case 'SYNC_PAIR_STAGE':
          // 修正要件：正準ペア値を受け取り、自分の視点で3本すべてを再描画する
          pairStage[data.payload.pairKey] = data.payload.stage;
          renderAllBarsForMe();
          break;
        case 'SYNC_PAIR_STAGE_ALL':
          // 修正要件：対戦開始時など、3ペア分をまとめて同期する
          pairStage = data.payload.pairStage;
          renderAllBarsForMe();
          break;
        case 'SYNC_SECONDARY_BAR':
          // 修正要件：2人モード専用、8段階の棒も対戦相手との有利不利を表す正準値として同期する
          secondaryPairStage = data.payload.stage;
          renderSecondaryBarForMe();
          break;
        case 'START_MULLIGAN':
          // 修正要件：先行プレイヤー決定後、ターン開始(デッキから1枚引く)前にマリガンを行う
          showMulliganModal();
          break;
        case 'MULLIGAN_CONFIRMED':
          // 修正要件：全員のマリガンが完了したら、ホストが先行プレイヤーのターンを開始する
          mulliganConfirmedCount++;
          if (isHost && pendingFirstPlayer !== null && mulliganConfirmedCount >= playerCount) {
            broadcast({ type: 'SET_TURN_PLAYER', payload: { index: pendingFirstPlayer } });
            pendingFirstPlayer = null;
          }
          break;
        case 'ACTION_LOG':
          logAction(data.payload.message);
          break;
        case 'CHAT':
          logAction(`${data.payload.name}: ${data.payload.message}`, true);
          break;
        case 'SET_TURN_PLAYER':
          applyTurnPlayer(data.payload.index);
          break;
        case 'GAME_OVER':
          showGameOverOverlay(data.payload.winnerIndex);
          break;
        case 'RESTART_GAME':
          performGameReset();
          break;
      }
    }

    function renderScoreBar() {
      const scoreBar = document.getElementById('score-bar');
      if (!scoreBar) return;

      // 修正要件：2人モードではプレイヤー3が存在しないため、剰余をplayerCountに合わせ、P3行を除外する
      let order = (playerCount === 2)
        ? [(myPlayerIndex + 1) % 2, myPlayerIndex]
        : [(myPlayerIndex + 1) % 3, myPlayerIndex, (myPlayerIndex + 2) % 3];

      let html = '';
      order.forEach((pIdx, idx) => {
        const pNum = pIdx + 1;
        const pName = playerNames[pIdx] || `P${pNum}`;
        const isSelf = (pIdx === myPlayerIndex);
        const isLast = (idx === order.length - 1);

        let titleClass = `score-p${pNum}-title`;
        let prefix = (idx === 0) ? '◀ ' : (isLast && !isSelf) ? '' : '★ ';
        let titleText = isSelf
          ? `${prefix}${pName} (あなた)`
          : `${prefix}${pName}`;

        if (idx === 0 && !isSelf) titleText = `◀ ${pName}`;
        if (isLast && !isSelf) titleText = `${pName} ▶`;

        html += `
          <div class="score-group">
            <span class="score-p-title ${titleClass}">${titleText}</span>
            <div class="score-box"><span class="score-label">トリニティ:</span><span id="p${pNum}-trinity-val" class="score-val">0</span></div>
            <div class="score-box"><span class="score-label">獲得ライフ:</span><span id="p${pNum}-card-score" class="score-val" style="color:#e0f2fe;">0</span></div>
            <div class="score-box">
              <span class="score-label">マナ:</span>
              <span id="p${pNum}-mana-count" class="score-val" style="color:#38bdf8;">0</span>
              <div class="mana-color-breakdown">
                (<span class="mc-red" id="p${pNum}-mana-red">0</span>/
                <span class="mc-yellow" id="p${pNum}-mana-yellow">0</span>/
                <span class="mc-blue" id="p${pNum}-mana-blue">0</span>/
                <span class="mc-purple" id="p${pNum}-mana-purple">0</span>)
              </div>
            </div>
          </div>
        `;
      });

      scoreBar.innerHTML = html;
      updateAllPlayerStatsDisplay();
    }

    function adjustPerspectiveBarLayout() {
      const trackLeftP1 = document.getElementById('track-left-p1');
      const trackTop = document.getElementById('track-top');
      const trackRightP1 = document.getElementById('track-right-p1');
      const table = document.getElementById('zone-table');

      if (!trackLeftP1 || !trackRightP1 || !trackTop || !table) return;

      [trackLeftP1, trackTop, trackRightP1].forEach(t => {
        t.classList.remove('bar-vertical', 'bar-flip');
      });

      if (myPlayerIndex === 1) {
        table.appendChild(trackTop);
        table.appendChild(trackLeftP1);
        table.appendChild(trackRightP1);

        trackTop.classList.add('bar-vertical', 'bar-flip');
        trackTop.style.position = 'absolute';
        trackTop.style.left = '20%';
        trackTop.style.right = 'auto';
        trackTop.style.bottom = '1vh';
        trackTop.style.top = 'auto';
        trackTop.style.width = '1.6vw';
        trackTop.style.height = '22vh';
        trackTop.style.transform = 'rotate(165deg)';

        trackLeftP1.classList.add('bar-vertical', 'bar-flip');
        trackLeftP1.style.position = 'absolute';
        trackLeftP1.style.right = '20%';
        trackLeftP1.style.left = 'auto';
        trackLeftP1.style.bottom = '1vh';
        trackLeftP1.style.top = 'auto';
        trackLeftP1.style.width = '1.6vw';
        trackLeftP1.style.height = '22vh';
        trackLeftP1.style.transform = 'rotate(-165deg)';

        trackRightP1.style.position = 'absolute';
        trackRightP1.style.top = '3vh';
        trackRightP1.style.left = '50%';
        trackRightP1.style.right = 'auto';
        trackRightP1.style.bottom = 'auto';
        trackRightP1.style.width = '15vw';
        trackRightP1.style.height = '3.2vh';
        trackRightP1.style.transform = 'translateX(-50%)';

      } else if (myPlayerIndex === 2) {
        table.appendChild(trackLeftP1);
        table.appendChild(trackRightP1);
        table.appendChild(trackTop);

        trackLeftP1.classList.add('bar-flip');
        trackLeftP1.style.position = 'absolute';
        trackLeftP1.style.top = '3vh';
        trackLeftP1.style.left = '50%';
        trackLeftP1.style.right = 'auto';
        trackLeftP1.style.bottom = 'auto';
        trackLeftP1.style.width = '15vw';
        trackLeftP1.style.height = '3.2vh';
        trackLeftP1.style.transform = 'translateX(-50%) rotate(180deg)';

        // 修正要件：P3視点は左右の棒を入れ替え（旧right-p1の配置をtrackTopへ、旧trackTopの配置をtrackRightP1へ）
        trackTop.classList.add('bar-vertical', 'bar-flip');
        trackTop.style.position = 'absolute';
        trackTop.style.left = '20%';
        trackTop.style.right = 'auto';
        trackTop.style.top = 'auto';
        trackTop.style.bottom = '1vh';
        trackTop.style.width = '1.6vw';
        trackTop.style.height = '22vh';
        trackTop.style.transform = 'rotate(165deg)';

        trackRightP1.classList.add('bar-vertical');
        trackRightP1.style.position = 'absolute';
        trackRightP1.style.right = '20%';
        trackRightP1.style.left = 'auto';
        trackRightP1.style.top = 'auto';
        trackRightP1.style.bottom = '1vh';
        trackRightP1.style.width = '1.6vw';
        trackRightP1.style.height = '22vh';
        trackRightP1.style.transform = 'rotate(15deg)';

      } else {
        table.appendChild(trackTop);
        table.appendChild(trackLeftP1);
        table.appendChild(trackRightP1);

        trackTop.style.position = 'absolute';
        trackTop.style.top = '3vh';
        trackTop.style.left = '50%';
        trackTop.style.right = 'auto';
        trackTop.style.bottom = 'auto';
        trackTop.style.width = '15vw';
        trackTop.style.height = '3.2vh';
        trackTop.style.transform = 'translateX(-50%)';

        trackLeftP1.classList.add('bar-vertical');
        trackLeftP1.style.position = 'absolute';
        trackLeftP1.style.bottom = '1vh';
        trackLeftP1.style.left = '20%';
        trackLeftP1.style.right = 'auto';
        trackLeftP1.style.top = 'auto';
        trackLeftP1.style.width = '1.6vw';
        trackLeftP1.style.height = '22vh';
        trackLeftP1.style.transform = 'rotate(-15deg)';

        trackRightP1.classList.add('bar-vertical');
        trackRightP1.style.position = 'absolute';
        trackRightP1.style.bottom = '1vh';
        trackRightP1.style.right = '20%';
        trackRightP1.style.left = 'auto';
        trackRightP1.style.top = 'auto';
        trackRightP1.style.width = '1.6vw';
        trackRightP1.style.height = '22vh';
        trackRightP1.style.transform = 'rotate(15deg)';
      }

      // 修正要件：DOM要素の再配置後は、現在の正準ペア値から自分の視点で正しく再描画する
      repositionMatchupLabels();
      hideUnusedBarsForTwoPlayerMode();

      // 修正要件：2人モードでは唯一表示される棒をまっすぐ縦向きにして右側へ移動する
      // (このあとのadjustPerspectiveBarLayout内の分岐でインラインstyleが設定済みのため、
      //  CSSではなくここで直接上書きする)
      if (playerCount === 2) {
        const usedPos = ['top', 'left-p1', 'right-p1'].find(pos => {
          const m = BAR_PAIR_MAP[myPlayerIndex][pos];
          return m && m.pairKey === 'AB';
        });
        const usedTrack = usedPos ? document.getElementById(`track-${usedPos}`) : null;
        if (usedTrack) {
          usedTrack.classList.add('bar-vertical');
          usedTrack.classList.remove('bar-flip');
          usedTrack.style.position = 'absolute';
          usedTrack.style.left = 'auto';
          usedTrack.style.right = '14%';
          usedTrack.style.top = 'auto';
          usedTrack.style.bottom = '1vh';
          usedTrack.style.width = '1.6vw';
          // 修正要件：棒をもう少し長くし、共有プレイエリアの中央あたりの高さまで届くようにする
          usedTrack.style.height = '26vh';
          usedTrack.style.transform = 'none';
        }
        // 修正要件：棒を右側へ動かしたのに合わせて、対応するラベルも右側へ追従させる
        // (repositionMatchupLabelsは通常視点の位置に配置済みのため、ここで上書きする)
        const usedLabel = usedPos ? document.getElementById(`matchup-${usedPos}`) : null;
        if (usedLabel) {
          usedLabel.style.bottom = '0.2vh';
          usedLabel.style.right = '6%';
          usedLabel.style.left = 'auto';
          usedLabel.style.top = 'auto';
          usedLabel.style.transform = 'none';
        }
      }

      if (typeof renderAllBarsForMe === 'function') {
        renderAllBarsForMe();
      } else {
        ['top', 'left-p1', 'right-p1'].forEach(pos => {
          updateBarStageLocal(pos, 1);
        });
      }
    }

    // 修正要件：2人モードでは対戦カードがP1vP2の1組しか存在しないため、
    // それ以外の(未使用の)棒とラベルを非表示にする
    function hideUnusedBarsForTwoPlayerMode() {
      if (playerCount !== 2) {
        ['top', 'left-p1', 'right-p1'].forEach(pos => {
          const track = document.getElementById(`track-${pos}`);
          const label = document.getElementById(`matchup-${pos}`);
          if (track) track.style.display = '';
          if (label) label.style.display = '';
        });
        return;
      }
      const mapping = BAR_PAIR_MAP[myPlayerIndex] || {};
      ['top', 'left-p1', 'right-p1'].forEach(pos => {
        const isUsed = mapping[pos] && mapping[pos].pairKey === 'AB';
        const track = document.getElementById(`track-${pos}`);
        const label = document.getElementById(`matchup-${pos}`);
        if (track) track.style.display = isUsed ? '' : 'none';
        if (label) label.style.display = isUsed ? '' : 'none';
      });
    }

    // 修正要件：adjustPerspectiveBarLayoutでDOM要素(track-top/left-p1/right-p1)が
    // 視点に応じて物理的に入れ替わるため、対応するラベルも同じ見た目の位置に追従させる
    const VISUAL_SLOT_FOR_DOM = {
      0: { 'left-p1': 'left', 'right-p1': 'right', 'top': 'top' },
      1: { 'top': 'left', 'left-p1': 'right', 'right-p1': 'top' },
      // 修正要件：P3視点は左右の棒を入れ替え
      2: { 'top': 'left', 'right-p1': 'right', 'left-p1': 'top' }
    };
    const LABEL_VISUAL_POS = {
      left: { bottom: '0.2vh', left: '6%', right: 'auto', top: 'auto', transform: 'none' },
      right: { bottom: '0.2vh', right: '6%', left: 'auto', top: 'auto', transform: 'none' },
      top: { top: '7.2vh', left: '50%', right: 'auto', bottom: 'auto', transform: 'translateX(-50%)' }
    };
    function repositionMatchupLabels() {
      const mapping = VISUAL_SLOT_FOR_DOM[myPlayerIndex] || VISUAL_SLOT_FOR_DOM[0];
      ['left-p1', 'right-p1', 'top'].forEach(domSuffix => {
        const el = document.getElementById(`matchup-${domSuffix}`);
        if (!el) return;
        const visualSlot = mapping[domSuffix];
        Object.assign(el.style, LABEL_VISUAL_POS[visualSlot]);
      });
    }

    /* 棒の操作関数 */
    // 修正要件：棒はプレイヤー同士のペアごとの有利不利を表す。
    // 正準値は3ペア分だけ持ち(AB=P1視点でのP1対P2, AC=P1視点でのP1対P3, BC=P2視点でのP2対P3)、
    // 各プレイヤーの画面では、そのプレイヤーの視点に応じて向き(反転)を変えて表示する。
    // 注意：adjustPerspectiveBarLayout()が視点に応じてtrack-top/left-p1/right-p1のDOM要素そのものを
    // 物理的に入れ替えて配置しているため（例：P2視点では見た目の「左」がDOM上は#track-top）、
    // ここでのマッピングはDOM要素ID基準で、実際の見た目の配置に合わせて対応させている。
    let pairStage = { AB: 1, AC: 1, BC: 1 };
    const BAR_PAIR_MAP = {
      0: { 'left-p1': { pairKey: 'AB', invert: false }, 'right-p1': { pairKey: 'AC', invert: false }, 'top': { pairKey: 'BC', invert: false } },
      1: { 'top': { pairKey: 'BC', invert: false }, 'left-p1': { pairKey: 'AB', invert: true }, 'right-p1': { pairKey: 'AC', invert: true } },
      // 修正要件：P3視点で左右の棒の内容を入れ替える(視覚上の左=P3vP1、右=P3vP2になるようにする)
      2: { 'top': { pairKey: 'AC', invert: true }, 'right-p1': { pairKey: 'BC', invert: true }, 'left-p1': { pairKey: 'AB', invert: false } }
    };

    // 修正要件：絶対プレイヤー番号2人と「Aから見た有利度(1-4)」を渡すだけで、
    // 正準ペア値(AB/AC/BC、必ず番号の小さい方基準)へ正しく変換して設定するヘルパー
    function setPairFavor(playerA, playerB, favorOfAStage) {
      const lo = Math.min(playerA, playerB);
      const hi = Math.max(playerA, playerB);
      const pairKey = (lo === 0 && hi === 1) ? 'AB' : (lo === 0 && hi === 2) ? 'AC' : 'BC';
      pairStage[pairKey] = (playerA === lo) ? favorOfAStage : (5 - favorOfAStage);
    }

    function setBarStage(pos, displayedStage) {
      const map = BAR_PAIR_MAP[myPlayerIndex][pos];
      if (!map) return;
      const canonicalStage = map.invert ? (5 - displayedStage) : displayedStage;
      pairStage[map.pairKey] = canonicalStage;
      broadcast({ type: 'SYNC_PAIR_STAGE', payload: { pairKey: map.pairKey, stage: canonicalStage } });
      renderAllBarsForMe();
    }

    // 修正要件：正準ペア値が更新されたら、自分の視点に応じて3本の棒すべてを再描画する
    function renderAllBarsForMe() {
      ['left-p1', 'right-p1', 'top'].forEach(pos => {
        const map = BAR_PAIR_MAP[myPlayerIndex][pos];
        const canonicalStage = pairStage[map.pairKey];
        const displayStage = map.invert ? (5 - canonicalStage) : canonicalStage;
        updateBarStageLocal(pos, displayStage);
      });
      updateMatchupLabels();
    }

    // 修正要件：各棒がどのプレイヤー同士の有利不利を示しているかをラベル表示する
    // (ラベルの左側の名前が有利=上/不利=下 の主語になる)
    const PAIR_PLAYERS = { AB: [0, 1], AC: [0, 2], BC: [1, 2] };
    function updateMatchupLabels() {
      ['left-p1', 'right-p1', 'top'].forEach(pos => {
        const map = BAR_PAIR_MAP[myPlayerIndex][pos];
        const el = document.getElementById(`matchup-${pos}`);
        if (!el || !map) return;
        let [a, b] = PAIR_PLAYERS[map.pairKey];
        if (map.invert) { const tmp = a; a = b; b = tmp; }
        const nameA = playerNames[a] || `P${a + 1}`;
        const nameB = playerNames[b] || `P${b + 1}`;
        el.innerText = `${nameA} vs ${nameB}`;
      });
    }

    // 修正要件：P2視点で「右の棒」は上下を反対に、「上部の棒」は左右を反対にする
    // P3視点は左右の棒(DOM: top, right-p1)を両方とも上下反対にする
    // (見た目の角度はそのまま。ハンドル位置の割合だけ反転させる)
    const BAR_REVERSED = {
      1: { 'left-p1': true, 'right-p1': true },
      2: { 'top': true, 'right-p1': true }
    };

    function updateBarStageLocal(pos, stage) {
      const handle = document.getElementById(`handle-${pos}`);
      const track = document.getElementById(`track-${pos}`);
      if (!handle || !track) return;

      const isVertical = track.classList.contains('bar-vertical');
      let percentage = (stage - 1) * 33.33;
      const reversed = BAR_REVERSED[myPlayerIndex] && BAR_REVERSED[myPlayerIndex][pos];
      if (reversed) percentage = 100 - percentage;

      if (isVertical) {
        handle.style.left = 'calc(50% - 1.9vw)';
        handle.style.top = `calc(${percentage}% - 1.25vh)`;
      } else {
        handle.style.left = `calc(${percentage}% - 0.75vw)`;
        handle.style.top = 'calc(50% - 3.25vh)';
      }
    }

    // 修正要件：2人モード専用、左側の8段階の棒も対戦相手との有利不利を表す。
    // 4段階の棒(pairStage)と同じ考え方で、絶対プレイヤー番号の小さい方基準の正準値(1-8)を持ち、
    // 各プレイヤーは自分の視点で(直接 or 9-N反転)表示する。
    // (プレイヤー1視点で下からN段階目 ⇔ プレイヤー2視点では上からN段階目、という対称性はこれで自動的に成立する)
    let secondaryPairStage = 1;

    function setSecondaryPairFavor(playerA, playerB, favorOfAStage) {
      const lo = Math.min(playerA, playerB);
      secondaryPairStage = (playerA === lo) ? favorOfAStage : (9 - favorOfAStage);
    }

    // 修正要件：クリック操作(自分視点での段階)を正準値に変換して同期する
    function setSecondaryBarStage(displayedStage) {
      const canonicalStage = (myPlayerIndex === 0) ? displayedStage : (9 - displayedStage);
      secondaryPairStage = canonicalStage;
      broadcast({ type: 'SYNC_SECONDARY_BAR', payload: { stage: canonicalStage } });
      renderSecondaryBarForMe();
    }

    // 修正要件：正準値から自分の視点での表示段階を計算して描画する
    function renderSecondaryBarForMe() {
      const displayedStage = (myPlayerIndex === 0) ? secondaryPairStage : (9 - secondaryPairStage);
      updateSecondaryBarStageLocal(displayedStage);
    }

    function updateSecondaryBarStageLocal(stage) {
      const handle = document.getElementById('handle-secondary-2p');
      if (!handle) return;
      const percentage = (stage - 1) * (100 / 7); // 8段階 = 7区間
      handle.style.left = 'calc(50% - 1.9vw)';
      handle.style.top = `calc(${percentage}% - 1.25vh)`;
    }

    /* カードプール */
    let masterDeck = [];
    let SPECIAL_CARDS = [];

    function shuffle(array) {
      for (let i = array.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [array[i], array[j]] = [array[j], array[i]];
      }
    }

    let poolCards = [], mainDeck = [], selectedSpecialCard = null;

    /* カード要素作成 */
    function createCardElement(card, onClick, sourceArea = null, draggableOverride = true) {
      const cardEl = document.createElement('div');
      cardEl.className = 'card-item';
      // 修正要件：自身の墓地一覧以外からは共有プレイエリアへドラッグできないようにする
      cardEl.draggable = draggableOverride;
      cardEl.innerHTML = `<img class="card-img-thumb" src="${card.img}" alt="card">`;
      
      if (onClick) cardEl.onclick = onClick;
      
      // 修正要件：ドラッグして共有エリアに出せるようデータにsource情報を含める
      if (draggableOverride) {
        cardEl.ondragstart = (e) => {
          e.dataTransfer.setData('text/plain', JSON.stringify({ card: card, sourceArea: sourceArea }));
        };
      } else {
        cardEl.style.cursor = 'default';
      }
      
      cardEl.onmouseenter = (e) => showPreview(card, e);
      cardEl.onmousemove = (e) => movePreview(e);
      cardEl.onmouseleave = () => hidePreview();
      return cardEl;
    }

    const previewEl = document.getElementById('hover-preview-card');
    function showPreview(card, event) {
      if (card && card.dummy) {
        previewEl.innerHTML = `<div class="preview-dummy-card">？</div>`;
      } else {
        previewEl.innerHTML = `<img id="pv-img" src="${card.img}" alt="Card Image">`;
      }
      previewEl.style.display = 'block';
      movePreview(event);
    }

    function movePreview(event) {
      const wrapper = document.getElementById('app-wrapper');
      const rect = wrapper.getBoundingClientRect();
      
      let left = event.clientX - rect.left + 20;
      let top = event.clientY - rect.top + 20;
      
      if (left + previewEl.offsetWidth > rect.width) left = event.clientX - rect.left - previewEl.offsetWidth - 10;
      if (top + previewEl.offsetHeight > rect.height) top = rect.height - previewEl.offsetHeight - 10;
      if (top < 10) top = 10;
      if (left < 10) left = 10;
      
      previewEl.style.left = `${left}px`;
      previewEl.style.top = `${top}px`;
    }

    function hidePreview() { previewEl.style.display = 'none'; }

    /* 対戦ゲームモード */
    let deckCards = [], lifeDecks = { left: [], right: [] }, currentRevealedLifeCard = null, targetLifeSlotIndex = null;
    let revealedCardSource = null; // 修正要件：'deck'(1枚めくる由来)か'life'(ライフ由来)かを区別
    let graveyards = [[], [], []], currentGyTab = 0, highestZIndex = 1000;
    let localManaList = [];
    const COLOR_ORDER = { 'red': 1, 'yellow': 2, 'blue': 3, 'purple': 4 };

    function createAllLockSlots() {
      ['p1', 'p2', 'p3'].forEach(pKey => {
        const zone = document.getElementById(`${pKey}-lock-zone`);
        zone.innerHTML = '';
        
        for (let i = 0; i < 21; i++) {
          const slot = document.createElement('div');
          slot.className = 'lock-slot';
          slot.setAttribute('data-slot-idx', i);

          if (pKey === 'p1') {
            if (i === 0) {
              slot.classList.add('life-slot');
              // 修正要件：自身の左ライフの残り枚数を表示
              slot.innerHTML = `<span>左L: <span id="own-left-life-count">-</span></span><button class="btn-life-flip" onclick="revealLifeCard('left')">めくる</button>`;
            } else if (i === 6) {
              slot.classList.add('life-slot');
              // 修正要件：自身の右ライフの残り枚数を表示
              slot.innerHTML = `<button class="btn-life-flip" onclick="revealLifeCard('right')">めくる</button><span>右L: <span id="own-right-life-count">-</span></span>`;
            } else if (i > 0 && i < 6) {
              slot.classList.add('empty-slot');
            } else if (i >= 7 && i <= 13) {
              slot.innerText = `${i - 6}`;
            } else if (i >= 14 && i <= 20) {
              slot.innerText = `${i - 6}`;
            }
          } else {
            /* 相手枠：左ライフ(i=0)と右ライフ(i=6)の表示 */
            if (i === 0) {
              slot.classList.add('life-slot');
              slot.innerText = '右ライフ';
            } else if (i === 6) {
              slot.classList.add('life-slot');
              slot.innerText = '左ライフ';
            } else if (i > 0 && i < 6) {
              slot.classList.add('empty-slot');
            } else if (i >= 7 && i <= 13) {
              slot.innerText = `${i - 6}`;
            } else if (i >= 14 && i <= 20) {
              slot.innerText = `${i - 6}`;
            }
          }
          zone.appendChild(slot);
        }
      });
      updateLockZoneColors();
    }

    function updateLockSlotsLifeText() {
      const p2Index = (myPlayerIndex + 1) % playerCount;
      const p2Zone = document.getElementById('p2-lock-zone');
      if (p2Zone) {
        const p2Slots = p2Zone.querySelectorAll('.lock-slot');
        if (p2Slots[0]) p2Slots[0].innerText = `左ライフ: ${playerStates[p2Index].leftLifeCount}`;
        if (p2Slots[6]) p2Slots[6].innerText = `右ライフ: ${playerStates[p2Index].rightLifeCount}`;
      }

      // 修正要件：2人モードにはプレイヤー3が存在しないため対象外にする
      if (playerCount === 3) {
        const p3Index = (myPlayerIndex + 2) % 3;
        const p3Zone = document.getElementById('p3-lock-zone');
        if (p3Zone) {
          const p3Slots = p3Zone.querySelectorAll('.lock-slot');
          if (p3Slots[0]) p3Slots[0].innerText = `左ライフ: ${playerStates[p3Index].leftLifeCount}`;
          if (p3Slots[6]) p3Slots[6].innerText = `右ライフ: ${playerStates[p3Index].rightLifeCount}`;
        }
      }
    }

    function getDisplayTarget(ownerIndex) {
      const relative = (ownerIndex - myPlayerIndex + playerCount) % playerCount;
      if (relative === 0) return { slotKey: 'p1', rotClass: null }; 
      if (relative === 1) return { slotKey: 'p2', rotClass: 'card-rot-p2' }; 
      return { slotKey: 'p3', rotClass: 'card-rot-p3' }; 
    }

    function getLockZoneElement(targetOwnerIndex) {
      const relative = (targetOwnerIndex - myPlayerIndex + playerCount) % playerCount;
      if (relative === 0) return { element: document.getElementById('p1-lock-zone'), rotClass: null };
      if (relative === 1) return { element: document.getElementById('p2-lock-zone'), rotClass: 'card-rot-p2' };
      return { element: document.getElementById('p3-lock-zone'), rotClass: 'card-rot-p3' };
    }

    // 修正要件：P1=赤/P2=水色/P3=緑のプレイヤー識別色
    const PLAYER_COLORS = ['#ef4444', '#22d3ee', '#22c55e'];

    function updateLockZoneColors() {
      for (let i = 0; i < playerCount; i++) {
        const target = getLockZoneElement(i);
        if (target.element) {
          target.element.classList.remove('lock-owner-0', 'lock-owner-1', 'lock-owner-2');
          target.element.classList.add(`lock-owner-${i}`);
        }
      }
    }

    /* 修正要件：ターンの概念 */
    let turnPlayerIndex = null;

    function applyTurnPlayer(index) {
      turnPlayerIndex = index;
      updateTurnPlayerHighlight();
      updateEndTurnButton();
      updateTurnRestrictedButtons();
      if (index === myPlayerIndex) {
        trinityChargeUsedThisTurn = false; // 修正要件：新しい自分のターンでチャージ可能に戻す
        showTurnAnnouncement();
        showTurnDrawModal();
      }
    }

    // 修正要件：セッション復帰/再接続時は「あなたのターンです」を出さずに状態だけ反映する
    function applyTurnPlayerSilent(index) {
      turnPlayerIndex = index;
      updateTurnPlayerHighlight();
      updateEndTurnButton();
      updateTurnRestrictedButtons();
    }

    // 修正要件：トリニティドロー/トリニティチャージ/検索は、ターンプレイヤーのみ操作可能にする
    // (獲得ライフに追加はドロップ先のため、別途ドロップ処理側でturnPlayerIndexチェックを行っている)
    function updateTurnRestrictedButtons() {
      const isMyTurn = turnPlayerIndex === myPlayerIndex;
      ['trinity-draw-btn', 'trinity-charge-btn', 'deck-search-btn'].forEach(id => {
        const btn = document.getElementById(id);
        if (!btn) return;
        btn.disabled = !isMyTurn;
        btn.style.opacity = isMyTurn ? '1' : '0.4';
        btn.style.cursor = isMyTurn ? 'pointer' : 'default';
      });
      // 獲得ライフに追加のドロップ枠も、見た目で分かるよう薄暗くする
      const scoreZoneEl = document.getElementById('score-add-zone');
      if (scoreZoneEl) scoreZoneEl.style.opacity = isMyTurn ? '1' : '0.4';
    }

    // 修正要件：ターン開始時、デッキの一番上のカードを画面中央に大きく表示。
    // デッキが0枚の場合は墓地をすべてデッキに戻してシャッフルし、上から3枚を並べて表示する。
    function showTurnDrawModal() {
      const modal = document.getElementById('turn-draw-modal');
      const title = document.getElementById('turn-draw-title');
      const container = document.getElementById('turn-draw-cards');
      if (!modal || !container) return;
      container.innerHTML = '';

      if (deckCards.length === 0) {
        if (graveyards[myPlayerIndex].length === 0) return; // デッキも墓地も無ければ何もしない

        deckCards = deckCards.concat(graveyards[myPlayerIndex]);
        graveyards[myPlayerIndex] = [];
        shuffle(deckCards);
        broadcastGraveyard(myPlayerIndex);
        updateDeckStatus();
        broadcastLog(`${myDisplayName()}の墓地がデッキに戻ってシャッフルされました`);

        // 修正要件：「どれか一枚を選択してください」ではなく、リフレッシュが起きたことを表示
        title.innerText = 'デッキが無かった為、リフレッシュしました';
        const drawn = deckCards.slice(-3).reverse();
        drawn.forEach(card => {
          const el = document.createElement('div');
          el.className = 'turn-draw-card';
          el.innerHTML = `<img src="${card.img}" alt="card">`;
          el.onclick = () => {
            drawn.forEach(c => {
              const idx = deckCards.indexOf(c);
              if (idx !== -1) deckCards.splice(idx, 1);
              addCardToHand(c);
            });
            updateDeckStatus();
            hideTurnDrawModal();
          };
          container.appendChild(el);
        });
      } else {
        title.innerText = 'ターンドロー（カードをクリックして手札へ）';
        const topCard = deckCards[deckCards.length - 1];
        const el = document.createElement('div');
        el.className = 'turn-draw-card single';
        el.innerHTML = `<img src="${topCard.img}" alt="card">`;
        el.onclick = () => {
          deckCards.pop();
          addCardToHand(topCard);
          updateDeckStatus();
          hideTurnDrawModal();
        };
        container.appendChild(el);
      }

      modal.style.display = 'flex';
    }

    function hideTurnDrawModal() {
      const modal = document.getElementById('turn-draw-modal');
      if (modal) modal.style.display = 'none';
    }

    function updateTurnPlayerHighlight() {
      ['p1', 'p2', 'p3'].forEach(k => {
        const zone = document.getElementById(`${k}-lock-zone`);
        if (zone) zone.classList.remove('turn-active');
      });
      if (turnPlayerIndex === null || turnPlayerIndex === undefined) return;
      const target = getDisplayTarget(turnPlayerIndex);
      const zone = document.getElementById(`${target.slotKey}-lock-zone`);
      if (zone) zone.classList.add('turn-active');
    }

    function updateEndTurnButton() {
      const btn = document.getElementById('end-turn-btn');
      if (!btn) return;
      const isMyTurn = turnPlayerIndex === myPlayerIndex;
      btn.disabled = !isMyTurn;
      btn.style.opacity = isMyTurn ? '1' : '0.4';
      btn.style.cursor = isMyTurn ? 'pointer' : 'default';
    }

    // 修正要件：新しいターンプレイヤーの固定枠に置かれたカードを全て縦向き(未タップ)に戻す
    function untapOwnerLockedCards(ownerIdx) {
      document.querySelectorAll('.placed-card').forEach(el => {
        const slotOwnerAttr = el.getAttribute('data-slot-owner');
        const isTapped = el.getAttribute('data-tapped') === 'true';
        if (slotOwnerAttr !== '' && parseInt(slotOwnerAttr) === ownerIdx && isTapped) {
          const cardData = {
            id: el.id, card: el.cardDataObj, ownerIndex: parseInt(el.getAttribute('data-owner')),
            x: parseFloat(el.style.left) || 0, y: parseFloat(el.style.top) || 0,
            faceDown: el.getAttribute('data-facedown') === 'true', tapped: false,
            slotOwnerIndex: ownerIdx, slotIndex: parseInt(el.getAttribute('data-slot-idx')),
            zIndex: parseInt(el.style.zIndex) || 10
          };
          broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
          updatePlacedCardDOM(el, cardData);
        }
      });
    }

    function endTurn() {
      if (turnPlayerIndex !== myPlayerIndex) return;
      const next = (turnPlayerIndex + 1) % playerCount;
      broadcast({ type: 'SET_TURN_PLAYER', payload: { index: next } });
      broadcastLog(`${myDisplayName()}がターンを終了しました`);
      untapOwnerLockedCards(next);
    }

    /* 修正要件：獲得ライフが12になったら勝敗を全員に通知する */
    let gameEnded = false;
    function checkWinCondition() {
      if (gameEnded) return;
      if ((playerStates[myPlayerIndex].lifeScore || 0) >= 12) {
        gameEnded = true;
        broadcast({ type: 'GAME_OVER', payload: { winnerIndex: myPlayerIndex } });
      }
    }

    function showGameOverOverlay(winnerIndex) {
      gameEnded = true;
      const overlay = document.getElementById('game-over-overlay');
      const text = document.getElementById('game-over-text');
      if (!overlay || !text) return;
      const isWinner = winnerIndex === myPlayerIndex;
      text.innerText = isWinner ? 'ゲーム終了！！あなたの勝利です！！' : 'ゲーム終了！！あなたは敗北しました....';
      text.style.color = isWinner ? '#fbbf24' : '#e2e8f0';
      overlay.style.display = 'flex';
      const restartBox = document.getElementById('restart-buttons');
      if (restartBox) restartBox.style.display = isHost ? 'flex' : 'none';
    }

    function hideGameOverOverlay() {
      const overlay = document.getElementById('game-over-overlay');
      if (overlay) overlay.style.display = 'none';
    }

    /* 修正要件：ホストが「ブレイズで再開」「ファントムで再開」を押すと、全員をリセットしてドラフトからやり直す */
    function restartGame(mode) {
      if (!isHost) return;
      broadcast({ type: 'RESTART_GAME', payload: {} });
      selectCardMode(mode);
    }

    function performGameReset() {
      gameEnded = false;
      hideGameOverOverlay();
      const restartBox = document.getElementById('restart-buttons');
      if (restartBox) restartBox.style.display = 'none';

      document.querySelectorAll('.placed-card').forEach(el => el.remove());
      const handContainer = document.getElementById('hand-cards');
      if (handContainer) handContainer.innerHTML = '';
      const manaZoneEl = document.getElementById('mana-zone');
      if (manaZoneEl) manaZoneEl.querySelectorAll('.mana-card').forEach(m => m.remove());

      // 修正要件：前回の対戦でピックしたカードが残らないよう、ドラフト/デッキ構築のDOM表示も明示的にクリアする
      // (picked-cardsのクリア漏れにより、リセット後も「獲得カード」の表示が残ってしまっていた)
      ['pack-cards', 'pool-cards', 'main-deck-cards', 'special-card-list', 'picked-cards'].forEach(id => {
        const el = document.getElementById(id);
        if (el) el.innerHTML = '';
      });
      const pickedCountEl = document.getElementById('picked-count');
      if (pickedCountEl) pickedCountEl.innerText = '0';

      deckCards = [];
      lifeDecks = { left: [], right: [] };
      graveyards = [[], [], []];
      localManaList = [];
      currentRevealedLifeCard = null;
      targetLifeSlotIndex = null;
      revealedCardSource = null;
      turnPlayerIndex = null;
      gameHasStarted = false;
      playerDeckReady = [false, false, false];
      pickedCards = [];
      poolCards = [];
      mainDeck = [];
      selectedSpecialCard = null;
      playerPickReady = [false, false, false];
      playerSelectedCards = [null, null, null];
      currentRound = 1;
      currentPick = 1;
      playerPacks = [[], [], []];
      roundPacksPool = [];
      selectedCard = null;
      // 修正要件：棒の正準値もリセットして次の対戦をニュートラルな状態から始める
      pairStage = { AB: 1, AC: 1, BC: 1 };

      playerStates = [
        { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 },
        { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 },
        { deckCount: 23, gyCount: 0, handCount: 0, lifeScore: 0, manaCount: 0, manaColors: { red: 0, yellow: 0, blue: 0, purple: 0 }, trinity: 0, leftLifeCount: 8, rightLifeCount: 8 }
      ];

      document.getElementById('battle-view').style.display = 'none';
      document.getElementById('deck-builder-view').style.display = 'none';
      document.getElementById('draft-view').style.display = 'flex';
      document.getElementById('header-bar').style.display = 'flex';
      document.getElementById('draft-status-info').style.display = 'flex';
      document.getElementById('deck-status-info').style.display = 'none';
      document.getElementById('phase-title').innerText = '🃏 ドラフトフェーズ';
    }

    function showTurnAnnouncement() {
      const el = document.getElementById('turn-announcement');
      if (!el) return;
      el.style.display = 'flex';
      requestAnimationFrame(() => el.classList.add('show'));
      setTimeout(() => {
        el.classList.remove('show');
        setTimeout(() => { el.style.display = 'none'; }, 300);
      }, 1000);
    }


    function getOwnerIndexFromZoneId(zoneId) {
      let relative = 0;
      if (zoneId === 'p1-lock-zone') relative = 0;
      if (zoneId === 'p2-lock-zone') relative = 1;
      if (zoneId === 'p3-lock-zone') relative = 2;
      // 修正要件：2人モードではプレイヤー3が存在しないため、剰余をplayerCountに合わせる
      return (myPlayerIndex + relative) % playerCount;
    }

    function findClosestSlot(clientX, clientY) {
      let closestSlot = null;
      let minDistance = Infinity;
      const SNAP_THRESHOLD = 90;

      const slots = document.querySelectorAll('.lock-slot:not(.empty-slot)');
      slots.forEach(slot => {
        const rect = slot.getBoundingClientRect();
        const slotCenterX = rect.left + rect.width / 2;
        const slotCenterY = rect.top + rect.height / 2;
        
        const dist = Math.hypot(clientX - slotCenterX, clientY - slotCenterY);
        if (dist < minDistance && dist <= SNAP_THRESHOLD) {
          minDistance = dist;
          const zoneEl = slot.parentElement;
          const slotIdx = parseInt(slot.getAttribute('data-slot-idx'));
          const slotOwnerIndex = getOwnerIndexFromZoneId(zoneEl.id);
          
          closestSlot = {
            zoneId: zoneEl.id,
            slotOwnerIndex: slotOwnerIndex,
            slotIndex: slotIdx
          };
        }
      });
      return closestSlot;
    }

    function getSlotPosition(slotOwnerIndex, slotIndex, cardId) {
      const table = document.getElementById('zone-table');
      const tableRect = table.getBoundingClientRect();
      
      const zoneInfo = getLockZoneElement(slotOwnerIndex);
      if (!zoneInfo || !zoneInfo.element) return null;

      const slots = zoneInfo.element.querySelectorAll('.lock-slot');
      const targetSlot = slots[slotIndex];
      if (!targetSlot) return null;

      const placedCards = Array.from(document.querySelectorAll(`.placed-card[data-slot-owner="${slotOwnerIndex}"][data-slot-idx="${slotIndex}"]`));
      // 修正要件：zIndexはクライアントごとに独立したローカルカウンタのため、視点によって重なり順(斜めオフセット)がズレていた。
      // 全員で必ず一致するカードID(生成時に払い出され、同期される)を基準に並べ替えることで、誰から見ても同じ並び順にする。
      placedCards.sort((a, b) => a.id.localeCompare(b.id));

      let stackIndex = placedCards.findIndex(el => el.id === cardId);
      if (stackIndex === -1) stackIndex = placedCards.length;

      const offset = stackIndex * 8;
      const slotRect = targetSlot.getBoundingClientRect();

      let targetCard = document.getElementById(cardId);
      let cardWidth = targetCard ? targetCard.offsetWidth : tableRect.width * 0.040;
      let cardHeight = targetCard ? targetCard.offsetHeight : tableRect.height * 0.100;

      const slotCenterX = slotRect.left + slotRect.width / 2 - tableRect.left;
      const slotCenterY = slotRect.top + slotRect.height / 2 - tableRect.top;

      return {
        x: slotCenterX - (cardWidth / 2) - offset,
        y: slotCenterY - (cardHeight / 2) - offset
      };
    }

    function startGame() {
      currentPhase = 'battle';
      document.getElementById('deck-builder-view').style.display = 'none';
      document.getElementById('battle-view').style.display = 'flex';
      document.getElementById('header-bar').style.display = 'none';
      // 修正要件：2人モードではP2を正面向きに、P3関連の表示を消すためのクラス切り替え
      document.getElementById('battle-view').classList.toggle('two-player-mode', playerCount === 2);

      document.getElementById('hand-cards').innerHTML = '';

      deckCards = [...mainDeck];
      shuffle(deckCards);

      const lifePool = [...poolCards];
      shuffle(lifePool);
      // 修正要件：2人モードは左ライフ4枚・右ライフ12枚の固定枚数に、3人モードは従来通り均等割り
      if (playerCount === 2) {
        lifeDecks.left = lifePool.slice(0, 4);
        lifeDecks.right = lifePool.slice(4, 16);
      } else {
        const half = Math.floor(lifePool.length / 2);
        lifeDecks.left = lifePool.slice(0, half);
        lifeDecks.right = lifePool.slice(half);
      }

      createAllLockSlots();

      if (selectedSpecialCard) {
        addCardToHand(selectedSpecialCard, true);
      }
      // 修正要件：固定カード1枚＋デッキの上から5枚の合計6枚でスタート
      for (let i = 0; i < 5; i++) {
        if (deckCards.length > 0) {
          addCardToHand(deckCards.pop());
        }
      }

      updateDeckStatus();
      updateLifeStatus();
      broadcastPlayerState();
    }

    function updateDeckStatus() {
      document.getElementById('deck-count').innerText = deckCards.length;
      playerStates[myPlayerIndex].deckCount = deckCards.length;
      broadcastPlayerState();
    }

    function updateLifeStatus() {
      playerStates[myPlayerIndex].leftLifeCount = lifeDecks.left.length;
      playerStates[myPlayerIndex].rightLifeCount = lifeDecks.right.length;
      // 修正要件：手前側(自分)のカード固定枠にも自身のライフ残り枚数を表示
      const leftEl = document.getElementById('own-left-life-count');
      if (leftEl) leftEl.innerText = lifeDecks.left.length;
      const rightEl = document.getElementById('own-right-life-count');
      if (rightEl) rightEl.innerText = lifeDecks.right.length;
      broadcastPlayerState();
    }

    function broadcastPlayerState() {
      const handCount = document.getElementById('hand-cards').children.length;
      playerStates[myPlayerIndex].handCount = handCount;
      playerStates[myPlayerIndex].gyCount = graveyards[myPlayerIndex].length;
      playerStates[myPlayerIndex].manaCount = localManaList.length;

      const colors = { red: 0, yellow: 0, blue: 0, purple: 0 };
      localManaList.forEach(m => {
        if (colors[m.color] !== undefined) colors[m.color]++;
      });
      playerStates[myPlayerIndex].manaColors = colors;

      broadcast({
        type: 'SYNC_PLAYER_STATE',
        payload: { index: myPlayerIndex, state: playerStates[myPlayerIndex] }
      });
      updateAllPlayerStatsDisplay();
    }

    function updateAllPlayerStatsDisplay() {
      for (let i = 0; i < playerCount; i++) {
        const pNum = i + 1;
        const st = playerStates[i];
        
        const scoreEl = document.getElementById(`p${pNum}-card-score`);
        if (scoreEl) scoreEl.innerText = st.lifeScore;

        const manaEl = document.getElementById(`p${pNum}-mana-count`);
        if (manaEl) manaEl.innerText = st.manaCount;

        const mColors = st.manaColors || { red: 0, yellow: 0, blue: 0, purple: 0 };
        const redEl = document.getElementById(`p${pNum}-mana-red`);
        if (redEl) redEl.innerText = mColors.red;
        const yellowEl = document.getElementById(`p${pNum}-mana-yellow`);
        if (yellowEl) yellowEl.innerText = mColors.yellow;
        const blueEl = document.getElementById(`p${pNum}-mana-blue`);
        if (blueEl) blueEl.innerText = mColors.blue;
        const purpleEl = document.getElementById(`p${pNum}-mana-purple`);
        if (purpleEl) purpleEl.innerText = mColors.purple;

        const trEl = document.getElementById(`p${pNum}-trinity-val`);
        if (trEl) trEl.innerText = st.trinity;
      }

      const p2Index = (myPlayerIndex + 1) % playerCount;

      const p2Title = document.getElementById('p2-overlay-title');
      p2Title.innerText = playerNames[p2Index] || `プレイヤー ${p2Index + 1}`;
      // 修正要件：相手情報の枠色をプレイヤー識別色(絶対番号基準)に合わせる
      p2Title.style.color = PLAYER_COLORS[p2Index];
      const p2Overlay = document.getElementById('p2-overlay-info');
      if (p2Overlay) p2Overlay.style.borderLeftColor = PLAYER_COLORS[p2Index];
      document.getElementById('p2-hand-count-overlay').innerText = playerStates[p2Index].handCount;
      document.getElementById('p2-deck-count-overlay').innerText = playerStates[p2Index].deckCount;
      document.getElementById('p2-gy-count-overlay').innerText = playerStates[p2Index].gyCount;

      // 修正要件：2人モードにはプレイヤー3が存在しないため関連表示を隠す
      const p3Overlay = document.getElementById('p3-overlay-info');
      if (playerCount === 3) {
        const p3Index = (myPlayerIndex + 2) % 3;
        if (p3Overlay) p3Overlay.style.display = '';
        const p3Title = document.getElementById('p3-overlay-title');
        p3Title.innerText = playerNames[p3Index] || `プレイヤー ${p3Index + 1}`;
        p3Title.style.color = PLAYER_COLORS[p3Index];
        if (p3Overlay) p3Overlay.style.borderRightColor = PLAYER_COLORS[p3Index];
        document.getElementById('p3-hand-count-overlay').innerText = playerStates[p3Index].handCount;
        document.getElementById('p3-deck-count-overlay').innerText = playerStates[p3Index].deckCount;
        document.getElementById('p3-gy-count-overlay').innerText = playerStates[p3Index].gyCount;
      } else if (p3Overlay) {
        p3Overlay.style.display = 'none';
      }

      for (let i = 0; i < 3; i++) {
        const btn = document.getElementById(`gy-tab-p${i}`);
        if (btn) {
          btn.innerText = playerNames[i] || `P${i + 1}`;
          btn.style.display = (i < playerCount) ? '' : 'none';
        }
      }

      updateLockSlotsLifeText();
    }

    function drawCard() {
      if (deckCards.length === 0) return alert('デッキが空です');
      const card = deckCards.pop();
      addCardToHand(card);
      updateDeckStatus();
      broadcastLog(`${myDisplayName()}がカードを1枚引きました`);
      recordUndo('カードを引く', () => {
        removeCardFromHandByRef(card);
        deckCards.push(card);
        updateDeckStatus();
      });
    }

    function revealTopDeckCard() {
      if (deckCards.length === 0) return alert('デッキが空です');
      currentRevealedLifeCard = deckCards.pop();
      targetLifeSlotIndex = null;
      revealedCardSource = 'deck';
      updateDeckStatus();
      // 修正要件：1枚めくるボタンを押したことをログで通知
      broadcastLog(`${myDisplayName()}がデッキの一番上のカードをめくりました`);

      document.getElementById('modal-card-img').src = currentRevealedLifeCard.img;
      document.getElementById('modal-card-actions').style.display = 'flex';
      document.getElementById('life-reveal-modal').style.display = 'flex';
    }

    // 修正要件：めくったカードをデッキの一番上/一番下に戻す
    function returnRevealedCardToDeckTop() {
      if (!currentRevealedLifeCard || revealedCardSource !== 'deck') return;
      deckCards.push(currentRevealedLifeCard);
      updateDeckStatus();
      broadcastLog(`${myDisplayName()}がめくったカードをデッキの一番上に戻しました`);
      currentRevealedLifeCard = null;
      targetLifeSlotIndex = null;
      revealedCardSource = null;
      document.getElementById('modal-card-actions').style.display = 'none';
      document.getElementById('life-reveal-modal').style.display = 'none';
    }

    function returnRevealedCardToDeckBottom() {
      if (!currentRevealedLifeCard || revealedCardSource !== 'deck') return;
      deckCards.unshift(currentRevealedLifeCard);
      updateDeckStatus();
      broadcastLog(`${myDisplayName()}がめくったカードをデッキの一番下に戻しました`);
      currentRevealedLifeCard = null;
      targetLifeSlotIndex = null;
      revealedCardSource = null;
      document.getElementById('modal-card-actions').style.display = 'none';
      document.getElementById('life-reveal-modal').style.display = 'none';
    }

    function addCardToHand(card, isFixed = false) {
      const container = document.getElementById('hand-cards');
      const wrapper = document.createElement('div');
      wrapper.className = 'hand-card-wrapper';
      // 修正要件：マリガン対象から固定カードを除外するための目印
      wrapper.dataset.fixed = isFixed ? 'true' : 'false';

      const cardEl = document.createElement('div');
      cardEl.className = 'dummy-card';
      cardEl.style.cursor = 'grab';
      cardEl.innerHTML = `<img src="${card.img}" alt="card">`;
      cardEl.__cardRef = card; // 修正要件：セッション復帰時の手札スナップショット用に参照を保持

      cardEl.onmouseenter = (e) => showPreview(card, e);
      cardEl.onmousemove = (e) => movePreview(e);
      cardEl.onmouseleave = () => hidePreview();

      cardEl.onmousedown = (e) => startHandCardDrag(e, card, wrapper, false);
      // 修正要件：裏で出す(右クリック)操作でブラウザ標準の右クリックメニューが出ないよう抑止
      cardEl.oncontextmenu = (e) => e.preventDefault();

      const faceDownBtn = document.createElement('button');
      faceDownBtn.className = 'btn-facedown';
      faceDownBtn.innerText = '裏で出す';
      faceDownBtn.onmousedown = (e) => {
        e.stopPropagation();
        startHandCardDrag(e, card, wrapper, true);
      };
      faceDownBtn.oncontextmenu = (e) => e.preventDefault();

      wrapper.appendChild(cardEl);
      wrapper.appendChild(faceDownBtn);
      container.appendChild(wrapper);
      broadcastPlayerState();
    }

    /* 修正要件：手札からのドラッグは、実際にドロップするまで他プレイヤーに一切共有しない。
       ドラッグ中はローカル専用の見た目(ゴースト)だけを追従させ、ドロップ結果が確定した瞬間に
       初めて共有プレイエリアのカードとして同期する。 */
    function startHandCardDrag(e, card, wrapperEl, forceFaceDown = false) {
      e.preventDefault();
      hidePreview();
      const table = document.getElementById('zone-table');
      const faceDownMode = forceFaceDown || (e.button === 2);
      const startX = e.clientX, startY = e.clientY;

      const ghost = document.createElement('div');
      ghost.className = 'dummy-card';
      ghost.style.position = 'fixed';
      ghost.style.pointerEvents = 'none';
      ghost.style.zIndex = '9999';
      ghost.style.opacity = '0.85';
      ghost.innerHTML = faceDownMode
        ? `<div style="width:100%;height:100%;background:#334155;display:flex;align-items:center;justify-content:center;color:#94a3b8;font-size:1vh;">裏面</div>`
        : `<img src="${card.img}" alt="card">`;
      document.body.appendChild(ghost);

      const gw = ghost.getBoundingClientRect().width;
      const gh = ghost.getBoundingClientRect().height;
      const positionGhost = (x, y) => {
        ghost.style.left = `${x - gw / 2}px`;
        ghost.style.top = `${y - gh / 2}px`;
      };
      positionGhost(startX, startY);

      let moved = false;

      const onMouseMove = (me) => {
        if (Math.hypot(me.clientX - startX, me.clientY - startY) > 4) moved = true;
        positionGhost(me.clientX, me.clientY);
      };

      const onMouseUp = (me) => {
        window.removeEventListener('mousemove', onMouseMove);
        window.removeEventListener('mouseup', onMouseUp);
        ghost.remove();

        if (!moved) return; // 単なるクリックは何もせず手札に留まる

        const inRect = (r) => r && me.clientX >= r.left && me.clientX <= r.right && me.clientY >= r.top && me.clientY <= r.bottom;

        const handPanel = document.getElementById('bottom-hand-panel').getBoundingClientRect();
        if (inRect(handPanel)) return; // 手札内に戻しただけ

        const scoreZone = document.getElementById('score-add-zone').getBoundingClientRect();
        const returnTopZoneEl = document.getElementById('deck-return-top-zone');
        const returnTopZone = returnTopZoneEl ? returnTopZoneEl.getBoundingClientRect() : null;
        const returnZone = document.getElementById('deck-return-zone').getBoundingClientRect();
        const returnBottomZone = document.getElementById('deck-bottom-zone').getBoundingClientRect();
        const gyZone = document.getElementById('graveyard-zone').getBoundingClientRect();

        if (inRect(scoreZone) && turnPlayerIndex === myPlayerIndex) {
          if (wrapperEl) wrapperEl.remove();
          playerStates[myPlayerIndex].lifeScore += 1;
          broadcastPlayerState();
          broadcastLog(`${myDisplayName()}が獲得ライフに追加しました`);
          checkWinCondition();
          // 修正要件：獲得ライフに加えた操作も一手戻せるようにする
          recordUndo('獲得ライフに追加', () => {
            playerStates[myPlayerIndex].lifeScore -= 1;
            broadcastPlayerState();
            addCardToHand(card);
          });
          return;
        }
        if (inRect(returnTopZone)) {
          if (wrapperEl) wrapperEl.remove();
          deckCards.push(card);
          updateDeckStatus();
          broadcastLog(`${myDisplayName()}がカードをデッキの一番上に戻しました`);
          return;
        }
        if (inRect(returnZone)) {
          if (wrapperEl) wrapperEl.remove();
          deckCards.push(card);
          shuffle(deckCards);
          updateDeckStatus();
          broadcastLog(`${myDisplayName()}がカードをデッキに戻しました`);
          return;
        }
        if (inRect(returnBottomZone)) {
          if (wrapperEl) wrapperEl.remove();
          deckCards.unshift(card);
          updateDeckStatus();
          broadcastLog(`${myDisplayName()}がカードをデッキの一番下に戻しました`);
          return;
        }
        if (inRect(gyZone)) {
          if (wrapperEl) wrapperEl.remove();
          graveyards[myPlayerIndex].push(card);
          broadcastGraveyard(myPlayerIndex);
          broadcastPlayerState();
          broadcastLog(`${myDisplayName()}がカードを墓地へ送りました`);
          return;
        }

        const tableRect = table.getBoundingClientRect();
        const insideTable = me.clientX >= tableRect.left && me.clientX <= tableRect.right && me.clientY >= tableRect.top && me.clientY <= tableRect.bottom;
        if (!insideTable) return; // 無効な場所 → 手札に留まる

        // 修正要件：ここで初めて共有プレイエリアのカードとして全員に共有する
        if (wrapperEl) wrapperEl.remove();
        broadcastPlayerState();

        const newCardId = 'bcard_' + Date.now() + '_' + Math.floor(Math.random() * 1000);
        let posX = me.clientX - tableRect.left - (tableRect.width * 0.020);
        let posY = me.clientY - tableRect.top - (tableRect.height * 0.050);
        let slotOwnerIndex = null, slotIndex = null;

        const closest = findClosestSlot(me.clientX, me.clientY);
        if (closest) {
          slotOwnerIndex = closest.slotOwnerIndex;
          slotIndex = closest.slotIndex;
          const slotPos = getSlotPosition(slotOwnerIndex, slotIndex, newCardId);
          if (slotPos) { posX = slotPos.x; posY = slotPos.y; }
        }

        highestZIndex++;
        const cardData = {
          id: newCardId, card: card, ownerIndex: myPlayerIndex,
          x: posX, y: posY, faceDown: faceDownMode, tapped: false,
          slotOwnerIndex, slotIndex, zIndex: highestZIndex
        };
        broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
        syncBoardCardLocal(cardData);
      };

      window.addEventListener('mousemove', onMouseMove);
      window.addEventListener('mouseup', onMouseUp);
    }

    /* 修正要件：デッキ検索・墓地から共有プレイエリアへドラッグ＆ドロップした際の受け入れ処理 */
    function handleTableDrop(e) {
      e.preventDefault();
      const dataStr = e.dataTransfer.getData('text/plain');
      if (!dataStr) return;
      try {
        const parsed = JSON.parse(dataStr);
        if (parsed.card && parsed.sourceArea) {
          const card = parsed.card;
          const source = parsed.sourceArea;

          if (source === 'deckSearch') {
            const idx = deckCards.findIndex(c => c.id === card.id);
            if (idx !== -1) deckCards.splice(idx, 1);
            updateDeckStatus();
            renderDeckSearchList();
          } else if (source === 'graveyard') {
            const list = graveyards[currentGyTab];
            const idx = list.findIndex(c => c.id === card.id);
            if (idx !== -1) list.splice(idx, 1);
            broadcastGraveyard(currentGyTab);
            broadcastPlayerState();
          }

          const table = document.getElementById('zone-table');
          const tableRect = table.getBoundingClientRect();
          const startX = e.clientX - tableRect.left - (tableRect.width * 0.020);
          const startY = e.clientY - tableRect.top - (tableRect.height * 0.050);

          const newCardId = 'bcard_' + Date.now() + '_' + Math.floor(Math.random() * 1000);
          const cardData = {
            id: newCardId,
            card: card,
            ownerIndex: myPlayerIndex,
            x: startX,
            y: startY,
            faceDown: false,
            tapped: false,
            slotOwnerIndex: null,
            slotIndex: null
          };

          broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
          syncBoardCardLocal(cardData);
        }
      } catch (err) {}
    }

    function syncBoardCardLocal(data) {
      let cardEl = document.getElementById(data.id);
      if (!cardEl) {
        cardEl = document.createElement('div');
        cardEl.id = data.id;
        cardEl.className = 'dummy-card placed-card';
        document.getElementById('zone-table').appendChild(cardEl);
        setupPlacedCardInteraction(cardEl, data);
      }
      updatePlacedCardDOM(cardEl, data);
    }

    function updatePlacedCardDOM(cardEl, data) {
      cardEl.setAttribute('data-owner', data.ownerIndex);
      cardEl.setAttribute('data-facedown', data.faceDown);
      cardEl.setAttribute('data-tapped', data.tapped);
      cardEl.setAttribute('data-slot-owner', data.slotOwnerIndex !== null ? data.slotOwnerIndex : '');
      cardEl.setAttribute('data-slot-idx', data.slotIndex !== null ? data.slotIndex : '');
      cardEl.cardDataObj = data.card;

      const targetInfo = getDisplayTarget(data.ownerIndex);
      
      cardEl.className = 'dummy-card placed-card orient-' + targetInfo.slotKey;
      if (targetInfo.rotClass) cardEl.classList.add(targetInfo.rotClass);
      if (data.faceDown) cardEl.classList.add('card-face-down');
      if (data.tapped) cardEl.classList.add('card-tapped');

      if (data.faceDown) {
        cardEl.innerHTML = `<div style="font-size:1.1vh; color:#64748b; margin-top:2vh;">裏面</div>`;
      } else {
        cardEl.innerHTML = `<img src="${data.card.img}" alt="card">`;
      }

      let posX = data.x;
      let posY = data.y;

      if (data.slotOwnerIndex !== null && data.slotIndex !== null) {
        const slotPos = getSlotPosition(data.slotOwnerIndex, data.slotIndex, data.id);
        if (slotPos) {
          posX = slotPos.x;
          posY = slotPos.y;
        }
      }

      cardEl.style.left = `${posX}px`;
      cardEl.style.top = `${posY}px`;
      cardEl.style.zIndex = data.zIndex || 10;
    }

    function setupPlacedCardInteraction(cardEl, initialData) {
      let isDragging = false;
      let startX, startY, origX, origY, origSlotOwner, origSlotIdx;
      let clickStartTime = 0;

      cardEl.oncontextmenu = (e) => e.preventDefault();

      cardEl.onmouseenter = (e) => {
        const isFaceDown = cardEl.getAttribute('data-facedown') === 'true';
        const ownerIdx = parseInt(cardEl.getAttribute('data-owner'));
        if (isFaceDown && ownerIdx !== myPlayerIndex) {
          showPreview({ dummy: true }, e);
        } else {
          showPreview(cardEl.cardDataObj, e);
        }
      };
      cardEl.onmousemove = (e) => movePreview(e);
      cardEl.onmouseleave = () => hidePreview();

      cardEl.onmousedown = (e) => {
        if (e.button !== 0) return;
        e.preventDefault();
        hidePreview(); // 修正要件：ドラッグ開始時に拡大プレビューが残らないよう非表示にする
        clickStartTime = Date.now();
        isDragging = true;
        highestZIndex++;
        cardEl.style.zIndex = highestZIndex;

        startX = e.clientX;
        startY = e.clientY;
        origX = parseFloat(cardEl.style.left) || 0;
        origY = parseFloat(cardEl.style.top) || 0;
        // 修正要件：元の位置に戻せるよう、ドラッグ開始時点のスロット情報も記録
        origSlotOwner = cardEl.getAttribute('data-slot-owner') !== '' ? parseInt(cardEl.getAttribute('data-slot-owner')) : null;
        origSlotIdx = cardEl.getAttribute('data-slot-idx') !== '' ? parseInt(cardEl.getAttribute('data-slot-idx')) : null;

        const onMouseMove = (me) => {
          if (!isDragging) return;
          const dx = me.clientX - startX;
          const dy = me.clientY - startY;
          const curX = origX + dx;
          const curY = origY + dy;

          cardEl.style.left = `${curX}px`;
          cardEl.style.top = `${curY}px`;

          const closest = findClosestSlot(me.clientX, me.clientY);
          document.querySelectorAll('.lock-slot:not(.empty-slot)').forEach(s => s.style.borderColor = s.classList.contains('life-slot') ? '#38bdf8' : '#475569');
          if (closest) {
            const zInfo = getLockZoneElement(closest.slotOwnerIndex);
            if (zInfo && zInfo.element) {
              const targetSlot = zInfo.element.querySelectorAll('.lock-slot')[closest.slotIndex];
              if (targetSlot) targetSlot.style.borderColor = '#f59e0b';
            }
          }
        };

        const onMouseUp = (me) => {
          if (!isDragging) return;
          isDragging = false;
          window.removeEventListener('mousemove', onMouseMove);
          window.removeEventListener('mouseup', onMouseUp);

          document.querySelectorAll('.lock-slot:not(.empty-slot)').forEach(s => s.style.borderColor = s.classList.contains('life-slot') ? '#38bdf8' : '#475569');

          const dragDuration = Date.now() - clickStartTime;
          const moveDist = Math.hypot(me.clientX - startX, me.clientY - startY);

          if (dragDuration < 200 && moveDist < 5) {
            const isFaceDown = cardEl.getAttribute('data-facedown') === 'true';
            const ownerIdx = parseInt(cardEl.getAttribute('data-owner'));
            const inLockSlot = cardEl.getAttribute('data-slot-owner') !== '';

            if (isFaceDown) {
              const cardData = {
                id: cardEl.id, card: cardEl.cardDataObj, ownerIndex: ownerIdx,
                x: origX, y: origY, faceDown: false,
                tapped: false,
                slotOwnerIndex: cardEl.getAttribute('data-slot-owner') !== '' ? parseInt(cardEl.getAttribute('data-slot-owner')) : null,
                slotIndex: cardEl.getAttribute('data-slot-idx') !== '' ? parseInt(cardEl.getAttribute('data-slot-idx')) : null,
                zIndex: highestZIndex
              };
              broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
              updatePlacedCardDOM(cardEl, cardData);
            } else if (inLockSlot) {
              const isTapped = cardEl.getAttribute('data-tapped') === 'true';
              const cardData = {
                id: cardEl.id, card: cardEl.cardDataObj, ownerIndex: ownerIdx,
                x: origX, y: origY, faceDown: false,
                tapped: !isTapped,
                slotOwnerIndex: parseInt(cardEl.getAttribute('data-slot-owner')),
                slotIndex: parseInt(cardEl.getAttribute('data-slot-idx')),
                zIndex: highestZIndex
              };
              broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
              updatePlacedCardDOM(cardEl, cardData);
            }
            return;
          }

          // 修正要件：Undo用に、ドロップ処理でカードが場から取り除かれる前の状態を記録
          const captureSnap = () => ({
            card: cardEl.cardDataObj,
            ownerIndex: parseInt(cardEl.getAttribute('data-owner')),
            faceDown: cardEl.getAttribute('data-facedown') === 'true',
            tapped: cardEl.getAttribute('data-tapped') === 'true',
            x: origX, y: origY,
            slotOwnerIndex: origSlotOwner, slotIndex: origSlotIdx
          });

          const handPanel = document.getElementById('bottom-hand-panel').getBoundingClientRect();
          if (me.clientX >= handPanel.left && me.clientX <= handPanel.right && me.clientY >= handPanel.top && me.clientY <= handPanel.bottom) {
            const snap = captureSnap();
            addCardToHand(cardEl.cardDataObj);
            removeBoardCard(cardEl.id);
            broadcastLog(`${myDisplayName()}がカードを手札に戻しました`);
            recordUndo('手札へ戻す', () => {
              removeCardFromHandByRef(snap.card);
              restoreCardToBoard(snap);
            });
            return;
          }

          const scoreZone = document.getElementById('score-add-zone').getBoundingClientRect();
          const returnZone = document.getElementById('deck-return-zone').getBoundingClientRect();
          const returnTopZoneEl = document.getElementById('deck-return-top-zone');
          const returnTopZone = returnTopZoneEl ? returnTopZoneEl.getBoundingClientRect() : null;
          const returnBottomZone = document.getElementById('deck-bottom-zone').getBoundingClientRect();
          const gyZone = document.getElementById('graveyard-zone').getBoundingClientRect();

          if (me.clientX >= scoreZone.left && me.clientX <= scoreZone.right && me.clientY >= scoreZone.top && me.clientY <= scoreZone.bottom && turnPlayerIndex === myPlayerIndex) {
            const snap = captureSnap();
            playerStates[myPlayerIndex].lifeScore += 1;
            removeBoardCard(cardEl.id);
            broadcastPlayerState();
            broadcastLog(`${myDisplayName()}が獲得ライフに追加しました`);
            recordUndo('獲得ライフに追加', () => {
              playerStates[myPlayerIndex].lifeScore -= 1;
              restoreCardToBoard(snap);
              broadcastPlayerState();
            });
            checkWinCondition();
            return;
          }

          if (me.clientX >= returnZone.left && me.clientX <= returnZone.right && me.clientY >= returnZone.top && me.clientY <= returnZone.bottom) {
            const snap = captureSnap();
            deckCards.push(cardEl.cardDataObj);
            shuffle(deckCards);
            removeBoardCard(cardEl.id);
            updateDeckStatus();
            broadcastLog(`${myDisplayName()}がカードをデッキに戻しました`);
            recordUndo('デッキに戻す', () => {
              const idx = deckCards.indexOf(snap.card);
              if (idx !== -1) deckCards.splice(idx, 1);
              updateDeckStatus();
              restoreCardToBoard(snap);
            });
            return;
          }

          if (returnTopZone && me.clientX >= returnTopZone.left && me.clientX <= returnTopZone.right && me.clientY >= returnTopZone.top && me.clientY <= returnTopZone.bottom) {
            const snap = captureSnap();
            deckCards.push(cardEl.cardDataObj);
            removeBoardCard(cardEl.id);
            updateDeckStatus();
            broadcastLog(`${myDisplayName()}がカードをデッキの一番上に戻しました`);
            recordUndo('デッキの一番上に戻す', () => {
              const idx = deckCards.indexOf(snap.card);
              if (idx !== -1) deckCards.splice(idx, 1);
              updateDeckStatus();
              restoreCardToBoard(snap);
            });
            return;
          }

          if (me.clientX >= returnBottomZone.left && me.clientX <= returnBottomZone.right && me.clientY >= returnBottomZone.top && me.clientY <= returnBottomZone.bottom) {
            const snap = captureSnap();
            deckCards.unshift(cardEl.cardDataObj);
            removeBoardCard(cardEl.id);
            updateDeckStatus();
            broadcastLog(`${myDisplayName()}がカードをデッキの一番下に戻しました`);
            recordUndo('デッキの一番下に戻す', () => {
              const idx = deckCards.indexOf(snap.card);
              if (idx !== -1) deckCards.splice(idx, 1);
              updateDeckStatus();
              restoreCardToBoard(snap);
            });
            return;
          }

          if (me.clientX >= gyZone.left && me.clientX <= gyZone.right && me.clientY >= gyZone.top && me.clientY <= gyZone.bottom) {
            const snap = captureSnap();
            graveyards[myPlayerIndex].push(cardEl.cardDataObj);
            removeBoardCard(cardEl.id);
            broadcastGraveyard(myPlayerIndex);
            broadcastPlayerState();
            broadcastLog(`${myDisplayName()}がカードを墓地へ送りました`);
            recordUndo('墓地へ送る', () => {
              const idx = graveyards[myPlayerIndex].indexOf(snap.card);
              if (idx !== -1) graveyards[myPlayerIndex].splice(idx, 1);
              broadcastGraveyard(myPlayerIndex);
              broadcastPlayerState();
              restoreCardToBoard(snap);
            });
            return;
          }

          /* 修正要件：手札・墓地・獲得ライフに追加・デッキに戻す・デッキの一番下に戻す以外の場所で、
             かつ共有プレイエリア(zone-table)の外にドロップされた場合は元の位置に戻す */
          const tableRectNow = document.getElementById('zone-table').getBoundingClientRect();
          const insideSharedArea = me.clientX >= tableRectNow.left && me.clientX <= tableRectNow.right && me.clientY >= tableRectNow.top && me.clientY <= tableRectNow.bottom;

          if (!insideSharedArea) {
            // 元々場に出ていたカードは元の位置（元のスロット）に戻す
            const revertData = {
              id: cardEl.id, x: origX, y: origY,
              slotOwnerIndex: origSlotOwner, slotIndex: origSlotIdx,
              zIndex: highestZIndex
            };
            broadcast({ type: 'MOVE_BOARD_CARD', payload: revertData });
            moveBoardCardLocal(revertData);
            return;
          }

          const closest = findClosestSlot(me.clientX, me.clientY);
          let finalSlotOwner = null;
          let finalSlotIdx = null;
          let finalX = origX + (me.clientX - startX);
          let finalY = origY + (me.clientY - startY);

          if (closest) {
            finalSlotOwner = closest.slotOwnerIndex;
            finalSlotIdx = closest.slotIndex;
          }

          broadcast({
            type: 'MOVE_BOARD_CARD',
            payload: {
              id: cardEl.id,
              x: finalX,
              y: finalY,
              slotOwnerIndex: finalSlotOwner,
              slotIndex: finalSlotIdx,
              zIndex: highestZIndex
            }
          });
          moveBoardCardLocal({
            id: cardEl.id,
            x: finalX,
            y: finalY,
            slotOwnerIndex: finalSlotOwner,
            slotIndex: finalSlotIdx,
            zIndex: highestZIndex
          });
        };

        window.addEventListener('mousemove', onMouseMove);
        window.addEventListener('mouseup', onMouseUp);
      };

      cardEl.ondblclick = () => {
        broadcast({ type: 'CHANGE_CARD_OWNER', payload: { id: cardEl.id, ownerIndex: myPlayerIndex } });
        changeCardOwnerLocal({ id: cardEl.id, ownerIndex: myPlayerIndex });
      };
    }

    function moveBoardCardLocal(data) {
      const cardEl = document.getElementById(data.id);
      if (!cardEl) return;
      cardEl.setAttribute('data-slot-owner', data.slotOwnerIndex !== null ? data.slotOwnerIndex : '');
      cardEl.setAttribute('data-slot-idx', data.slotIndex !== null ? data.slotIndex : '');
      cardEl.style.zIndex = data.zIndex || 10;

      let posX = data.x;
      let posY = data.y;

      if (data.slotOwnerIndex !== null && data.slotIndex !== null) {
        const slotPos = getSlotPosition(data.slotOwnerIndex, data.slotIndex, data.id);
        if (slotPos) {
          posX = slotPos.x;
          posY = slotPos.y;
        }
      }

      cardEl.style.left = `${posX}px`;
      cardEl.style.top = `${posY}px`;
    }

    function changeCardOwnerLocal(data) {
      const cardEl = document.getElementById(data.id);
      if (!cardEl) return;
      cardEl.setAttribute('data-owner', data.ownerIndex);
      const targetInfo = getDisplayTarget(data.ownerIndex);
      
      cardEl.className = 'dummy-card placed-card orient-' + targetInfo.slotKey;
      if (targetInfo.rotClass) cardEl.classList.add(targetInfo.rotClass);
      if (cardEl.getAttribute('data-facedown') === 'true') cardEl.classList.add('card-face-down');
      if (cardEl.getAttribute('data-tapped') === 'true') cardEl.classList.add('card-tapped');
    }

    function removeBoardCard(id) {
      broadcast({ type: 'REMOVE_BOARD_CARD', payload: { id: id } });
      removeBoardCardLocal(id);
    }

    function removeBoardCardLocal(id) {
      const cardEl = document.getElementById(id);
      if (cardEl) {
        // 修正要件：カードが場から消える際、拡大プレビューが表示されたままにならないよう保険で閉じる
        hidePreview();
        cardEl.remove();
      }
    }

    /* 修正要件：アクションログ／簡易チャット */
    function myDisplayName() {
      return playerNames[myPlayerIndex] || myName || `P${myPlayerIndex + 1}`;
    }

    function manaColorLabel(color) {
      return { red: '赤', yellow: '黄', blue: '青', purple: '紫' }[color] || color;
    }

    function broadcastLog(msg) {
      broadcast({ type: 'ACTION_LOG', payload: { message: msg } });
    }

    function logAction(msg, isChat = false) {
      const container = document.getElementById('action-log-toast');
      if (!container) return;
      const item = document.createElement('div');
      item.className = 'action-log-item' + (isChat ? ' chat' : '');
      item.innerText = msg;
      container.appendChild(item);
      requestAnimationFrame(() => item.classList.add('show'));
      setTimeout(() => {
        item.classList.remove('show');
        setTimeout(() => item.remove(), 350);
      }, 3200);
      while (container.children.length > 4) {
        container.removeChild(container.firstChild);
      }
    }

    function sendChatMessage() {
      const input = document.getElementById('chat-input');
      if (!input) return;
      const msg = input.value.trim();
      if (!msg) return;
      input.value = '';
      broadcast({ type: 'CHAT', payload: { name: myDisplayName(), message: msg } });
    }

    /* 修正要件：直前操作の取り消し(Undo) */
    let lastUndoAction = null;
    let undoHideTimer = null;

    function recordUndo(label, undoFn) {
      lastUndoAction = { label, undoFn };
      const btn = document.getElementById('undo-btn');
      if (btn) {
        btn.innerText = `元に戻す（${label}）`;
        btn.style.display = 'block';
      }
      clearTimeout(undoHideTimer);
      undoHideTimer = setTimeout(() => clearUndo(), 15000);
    }

    function clearUndo() {
      lastUndoAction = null;
      const btn = document.getElementById('undo-btn');
      if (btn) btn.style.display = 'none';
    }

    function undoLastAction() {
      if (!lastUndoAction) return;
      const action = lastUndoAction;
      clearUndo();
      action.undoFn();
      logAction(`「${action.label}」を元に戻しました`);
    }

    function removeCardFromHandByRef(cardRef) {
      const handContainer = document.getElementById('hand-cards');
      if (!handContainer) return;
      const wrappers = Array.from(handContainer.children);
      for (let i = wrappers.length - 1; i >= 0; i--) {
        const cEl = wrappers[i].querySelector('.dummy-card');
        if (cEl && cEl.__cardRef === cardRef) {
          wrappers[i].remove();
          break;
        }
      }
      broadcastPlayerState();
    }

    function restoreCardToBoard(snap) {
      const newId = 'bcard_' + Date.now() + '_' + Math.floor(Math.random() * 1000);
      highestZIndex++;
      const cardData = {
        id: newId, card: snap.card, ownerIndex: snap.ownerIndex,
        x: snap.x, y: snap.y, faceDown: snap.faceDown, tapped: snap.tapped,
        slotOwnerIndex: snap.slotOwnerIndex, slotIndex: snap.slotIndex, zIndex: highestZIndex
      };
      broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
      syncBoardCardLocal(cardData);
    }

    function revealLifeCard(side) {
      const pool = lifeDecks[side];
      if (!pool || pool.length === 0) return alert(`${side === 'left' ? '左' : '右'}のライフデッキが空です`);
      currentRevealedLifeCard = pool.pop();
      targetLifeSlotIndex = (side === 'left') ? 0 : 6;
      revealedCardSource = 'life';
      updateLifeStatus();

      // 修正要件：ライフデッキが0枚になった瞬間にパニッシュバーンが発生する
      // (右のライフデッキが尽きれば右の棒、左が尽きれば左の棒がそのプレイヤーにとって有利になる)
      // 注意：「左/右」は常に自分の視点での左右バーを指すため、DOM要素ID経由(setBarStage)ではなく、
      // 対戦カードを直接指定するsetPairFavorで設定する(視点によってDOM要素と見た目の左右が入れ替わるため)
      if (pool.length === 0) {
        // 修正要件：2人モードは対戦相手が1人だけのため、左右どちらのライフデッキが尽きても
        // 唯一の対戦相手との対戦カードを有利にする（3人モードは従来通り左右で相手を分ける）
        const opponent = (playerCount === 2)
          ? (myPlayerIndex + 1) % 2
          : (side === 'left' ? (myPlayerIndex + 1) % 3 : (myPlayerIndex + 2) % 3);
        setPairFavor(myPlayerIndex, opponent, 1);
        broadcast({ type: 'SYNC_PAIR_STAGE_ALL', payload: { pairStage: Object.assign({}, pairStage) } });
        renderAllBarsForMe();
        broadcastLog(`${myDisplayName()}の${side === 'left' ? '左' : '右'}のライフデッキが尽き、パニッシュバーンが発生しました！`);
      }

      document.getElementById('modal-card-img').src = currentRevealedLifeCard.img;
      document.getElementById('modal-card-actions').style.display = 'none';
      document.getElementById('life-reveal-modal').style.display = 'flex';
    }

    function placeLifeCardToField() {
      if (!currentRevealedLifeCard) return;
      document.getElementById('life-reveal-modal').style.display = 'none';
      document.getElementById('modal-card-actions').style.display = 'none';

      const table = document.getElementById('zone-table');
      const tableRect = table.getBoundingClientRect();
      const centerX = tableRect.width / 2 - (tableRect.width * 0.020);
      const centerY = tableRect.height / 2 - (tableRect.height * 0.050);

      const newCardId = 'bcard_' + Date.now() + '_' + Math.floor(Math.random() * 1000);
      const cardData = {
        id: newCardId,
        card: currentRevealedLifeCard,
        ownerIndex: myPlayerIndex,
        x: centerX,
        y: centerY,
        faceDown: false,
        tapped: false,
        slotOwnerIndex: targetLifeSlotIndex !== null ? myPlayerIndex : null,
        slotIndex: targetLifeSlotIndex
      };

      currentRevealedLifeCard = null;
      targetLifeSlotIndex = null;

      broadcast({ type: 'SYNC_BOARD_CARD', payload: cardData });
      syncBoardCardLocal(cardData);
    }

    function broadcastGraveyard(pIdx) {
      broadcast({
        type: 'SYNC_GRAVEYARD',
        payload: { ownerIndex: pIdx, graveyard: graveyards[pIdx] }
      });
      renderGraveyardList();
    }

    function switchGyTab(pIdx) {
      currentGyTab = pIdx;
      for (let i = 0; i < 3; i++) {
        const btn = document.getElementById(`gy-tab-p${i}`);
        if (btn) btn.style.background = (i === pIdx) ? '#4f46e5' : '#334155';
      }
      renderGraveyardList();
    }

    /* 修正要件：墓地一覧のカードクリックで手札に入らないようonClickをnullにし、ドラッグ可能化 */
    function renderGraveyardList() {
      const container = document.getElementById('graveyard-list');
      if (!container) return;
      container.innerHTML = '';

      const list = graveyards[currentGyTab] || [];
      document.getElementById('gy-count').innerText = graveyards[myPlayerIndex].length;

      list.forEach((card) => {
        // 修正要件：自身の墓地一覧を表示している時だけドラッグ可能にする
        const cardEl = createCardElement(card, null, 'graveyard', currentGyTab === myPlayerIndex);
        container.appendChild(cardEl);
      });
    }

    /* 修正要件：デッキ内検索画面を手札・マナエリアを覆うオーバーレイに変更 */
    function toggleDeckSearch() {
      const overlay = document.getElementById('deck-search-overlay');
      if (overlay.style.display === 'none' || overlay.style.display === '') {
        // 修正要件：デッキ内検索を開く操作はターンプレイヤーのみ（閉じる操作は誰でも可能）
        if (turnPlayerIndex !== myPlayerIndex) return;
        overlay.style.display = 'flex';
        renderDeckSearchList();
      } else {
        overlay.style.display = 'none';
      }
    }

    /* 修正要件：デッキ内検索のカードクリックで手札に入らないようonClickをnullにし、ドラッグ可能化 */
    function renderDeckSearchList() {
      const container = document.getElementById('deck-search-list-horizontal');
      if (!container) return;
      container.innerHTML = '';

      let searchList = [...deckCards];
      shuffle(searchList);

      searchList.forEach((card) => {
        const cardEl = createCardElement(card, null, 'deckSearch');
        container.appendChild(cardEl);
      });
    }

    function spawnMana(color) {
      const manaObj = { id: 'mana_' + Date.now() + '_' + Math.floor(Math.random()*1000), color: color, tapped: false };
      localManaList.push(manaObj);
      renderManaZone();
      broadcastPlayerState();
    }

    function renderManaZone() {
      const zone = document.getElementById('mana-zone');
      zone.querySelectorAll('.mana-card').forEach(m => m.remove());

      localManaList.sort((a, b) => COLOR_ORDER[a.color] - COLOR_ORDER[b.color]);

      localManaList.forEach((mana, idx) => {
        const manaEl = document.createElement('div');
        manaEl.id = mana.id;
        manaEl.className = `mana-card mana-${mana.color}` + (mana.tapped ? ' tapped' : '');
        manaEl.innerText = mana.color.charAt(0).toUpperCase();

        // 修正要件：マナ同士の重なりを少しだけ小さく(1.2vw -> 1.5vw間隔)
        const overlapOffset = idx * 1.5;
        manaEl.style.left = `calc(10vw + ${overlapOffset}vw)`;

        let isDragging = false;
        let startX = 0, startY = 0;
        let origLeft = 0, origTop = 0;

        manaEl.onmousedown = (e) => {
          if (e.button !== 0) return;
          e.preventDefault();
          e.stopPropagation();

          isDragging = false;
          startX = e.clientX;
          startY = e.clientY;

          const rect = manaEl.getBoundingClientRect();
          const parentRect = zone.getBoundingClientRect();
          origLeft = rect.left - parentRect.left;
          origTop = rect.top - parentRect.top;

          const onMouseMove = (me) => {
            const dx = me.clientX - startX;
            const dy = me.clientY - startY;

            if (Math.hypot(dx, dy) > 3) {
              if (!isDragging) {
                isDragging = true;
                manaEl.style.zIndex = '1000';
              }
              manaEl.style.left = `${origLeft + dx}px`;
              manaEl.style.top = `${origTop + dy}px`;
            }
          };

          const onMouseUp = (me) => {
            window.removeEventListener('mousemove', onMouseMove);
            window.removeEventListener('mouseup', onMouseUp);

            if (isDragging) {
              const zoneRect = zone.getBoundingClientRect();
              if (me.clientX < zoneRect.left || me.clientX > zoneRect.right || me.clientY < zoneRect.top || me.clientY > zoneRect.bottom) {
                // 修正要件：マナをドラッグしてエリア外に破棄した際、誰が何色のマナを破棄したかをログ通知
                localManaList = localManaList.filter(m => m.id !== mana.id);
                renderManaZone();
                broadcastPlayerState();
                broadcastLog(`${myDisplayName()}が${manaColorLabel(mana.color)}マナを破棄しました`);
              } else {
                renderManaZone();
              }
            } else {
              // 修正要件：マナを横向き(使用)にした際、誰が何色のマナを使用したかをログ通知
              mana.tapped = !mana.tapped;
              renderManaZone();
              if (mana.tapped) {
                broadcastLog(`${myDisplayName()}が${manaColorLabel(mana.color)}マナを使用しました`);
              }
              // 修正要件：トリニティチャージ中は、新たに横にしたマナの枚数を追跡する
              if (trinityChargeActive) {
                if (mana.tapped) {
                  trinityChargeNewlyTapped.add(mana.id);
                } else {
                  trinityChargeNewlyTapped.delete(mana.id);
                }
                updateTrinityChargeConfirmVisibility();
              }
            }
          };

          window.addEventListener('mousemove', onMouseMove);
          window.addEventListener('mouseup', onMouseUp);
        };

        zone.appendChild(manaEl);
      });
    }

    function untapAllMana() {
      localManaList.forEach(m => m.tapped = false);
      renderManaZone();
    }

    /* 修正要件：トリニティカウンターの上限を10に修正 */
    function updateTrinity(val) {
      const current = playerStates[myPlayerIndex].trinity || 0;
      playerStates[myPlayerIndex].trinity = Math.min(10, Math.max(0, current + val));
      document.getElementById('counter-num').innerText = playerStates[myPlayerIndex].trinity;
      broadcastPlayerState();
    }

    /* 修正要件：トリニティドロー（トリニティ3消費でデッキ一番上をドロー。誤操作防止のため確認あり） */
    function requestTrinityDraw() {
      // 修正要件：ターンプレイヤーのみ操作可能
      if (turnPlayerIndex !== myPlayerIndex) return;
      const trinity = playerStates[myPlayerIndex].trinity || 0;
      if (trinity < 3) return alert('トリニティが3つ未満のため実行できません');
      if (deckCards.length === 0) return alert('デッキが空です');
      if (!confirm('トリニティを3つ消費して、デッキの一番上のカードをドローします。よろしいですか？')) return;
      playerStates[myPlayerIndex].trinity = trinity - 3;
      document.getElementById('counter-num').innerText = playerStates[myPlayerIndex].trinity;
      const card = deckCards.pop();
      addCardToHand(card);
      updateDeckStatus();
      broadcastPlayerState();
      broadcastLog(`${myDisplayName()}がトリニティドローを行いました`);
    }

    /* 修正要件：トリニティチャージ（マナゾーン以外を暗転させ、新たに3枚横向きにすると決定ボタンが出現） */
    let trinityChargeActive = false;
    let trinityChargeNewlyTapped = new Set();
    let trinityChargeUsedThisTurn = false; // 修正要件：1ターンに1度のみ

    function startTrinityCharge() {
      // 修正要件：ターンプレイヤーのみ操作可能、かつ1ターンに1度のみ
      if (turnPlayerIndex !== myPlayerIndex) return;
      if (trinityChargeUsedThisTurn) return alert('トリニティチャージはこのターンで既に行いました');
      if (trinityChargeActive) return;
      trinityChargeActive = true;
      trinityChargeNewlyTapped = new Set();
      document.getElementById('trinity-charge-dim').style.display = 'block';
      document.getElementById('mana-zone').classList.add('trinity-charge-active');
      document.getElementById('untap-all-btn').style.display = 'none';
      document.getElementById('trinity-charge-cancel-btn').style.display = 'block';
      document.getElementById('trinity-charge-confirm-btn').style.display = 'none';
    }

    function updateTrinityChargeConfirmVisibility() {
      const btn = document.getElementById('trinity-charge-confirm-btn');
      if (!btn) return;
      btn.style.display = (trinityChargeNewlyTapped.size === 3) ? 'block' : 'none';
    }

    function cancelTrinityCharge() {
      if (!trinityChargeActive) return;
      // 新しく横にしたマナを縦向きに戻す
      localManaList.forEach(m => {
        if (trinityChargeNewlyTapped.has(m.id)) m.tapped = false;
      });
      renderManaZone();
      endTrinityChargeUI();
    }

    function confirmTrinityCharge() {
      if (!trinityChargeActive || trinityChargeNewlyTapped.size !== 3) return;
      const current = playerStates[myPlayerIndex].trinity || 0;
      playerStates[myPlayerIndex].trinity = Math.min(10, current + 1);
      document.getElementById('counter-num').innerText = playerStates[myPlayerIndex].trinity;
      broadcastPlayerState();
      broadcastLog(`${myDisplayName()}がトリニティチャージを行いました`);
      trinityChargeUsedThisTurn = true; // 修正要件：1ターンに1度のみ
      endTrinityChargeUI();
    }

    function endTrinityChargeUI() {
      trinityChargeActive = false;
      trinityChargeNewlyTapped = new Set();
      document.getElementById('trinity-charge-dim').style.display = 'none';
      document.getElementById('mana-zone').classList.remove('trinity-charge-active');
      document.getElementById('untap-all-btn').style.display = 'block';
      document.getElementById('trinity-charge-cancel-btn').style.display = 'none';
      document.getElementById('trinity-charge-confirm-btn').style.display = 'none';
    }
