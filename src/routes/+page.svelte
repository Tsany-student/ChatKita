<script>
  import { browser } from "$app/environment";
  import { goto } from "$app/navigation";
  import { supabase } from "$lib/supabase";

  let user = $state(null);
  let users = $state([]);
  let selectedUser = $state(null);
  let messages = $state([]);
  let newMessage = $state("");
  let loading = $state(true);
  let errorMessage = $state("");

  let realtimeChannel = null;

  // =========================
  // LOAD USER
  // =========================

  async function loadUser() {
    loading = true;
    errorMessage = "";

    try {
      const {
        data: { session },
        error
      } = await supabase.auth.getSession();

      console.log("SESSION:", session);
      console.log("SESSION ERROR:", error);

      if (error) {
        throw error;
      }

      // Belum login
      if (!session) {
        console.log("User belum login");

        await goto("/");
        return;
      }

      user = session.user;

      console.log("USER LOGIN:", user);

      // Load user lain
      await loadUsers();

      // Setup realtime setelah user diketahui
      setupRealtime();

    } catch (error) {
      console.error("LOAD USER ERROR:", error);

      errorMessage =
        error?.message || "Terjadi kesalahan saat memuat aplikasi.";

    } finally {
      loading = false;
    }
  }

  // =========================
  // LOAD USERS
  // =========================

  async function loadUsers() {
    if (!user) return;

    const { data, error } = await supabase
      .from("profiles")
      .select("*")
      .neq("id", user.id)
      .order("username", { ascending: true });

    console.log("PROFILES:", data);
    console.log("PROFILES ERROR:", error);

    if (error) {
      throw error;
    }

    users = data || [];
  }

  // =========================
  // OPEN CHAT
  // =========================

  async function openChat(person) {
    if (!user) return;

    selectedUser = person;
    messages = [];

    console.log("OPEN CHAT:", person);

    const { data, error } = await supabase
      .from("messages")
      .select("*")
      .or(
        `and(sender_id.eq.${user.id},receiver_id.eq.${person.id}),and(sender_id.eq.${person.id},receiver_id.eq.${user.id})`
      )
      .order("created_at", {
        ascending: true
      });

    if (error) {
      console.error("LOAD MESSAGES ERROR:", error);

      errorMessage = error.message;

      return;
    }

    messages = data || [];

    console.log("MESSAGES:", messages);
  }

  // =========================
  // SEND MESSAGE
  // =========================

  async function sendMessage() {
    if (!user) return;

    if (!selectedUser) {
      alert("Pilih pengguna terlebih dahulu.");
      return;
    }

    const text = newMessage.trim();

    if (!text) return;

    newMessage = "";

    const { data, error } = await supabase
      .from("messages")
      .insert({
        sender_id: user.id,
        receiver_id: selectedUser.id,
        content: text
      })
      .select()
      .single();

    if (error) {
      console.error("SEND MESSAGE ERROR:", error);

      alert(error.message);

      newMessage = text;

      return;
    }

    console.log("MESSAGE SENT:", data);

    /*
      Jangan push ke messages di sini.

      Realtime akan menerima message INSERT
      dan memasukkannya otomatis.
    */
  }

  // =========================
  // REALTIME
  // =========================

  function setupRealtime() {
    if (!user) return;

    // Hapus channel lama kalau ada
    if (realtimeChannel) {
      supabase.removeChannel(realtimeChannel);
    }

    realtimeChannel = supabase
      .channel(`messages-${user.id}`)

      .on(
        "postgres_changes",
        {
          event: "INSERT",
          schema: "public",
          table: "messages"
        },
        (payload) => {
          const message = payload.new;

          console.log("REALTIME MESSAGE:", message);

          if (!selectedUser) return;

          const isCurrentChat =
            (
              message.sender_id === user.id &&
              message.receiver_id === selectedUser.id
            ) ||
            (
              message.sender_id === selectedUser.id &&
              message.receiver_id === user.id
            );

          if (!isCurrentChat) return;

          // Hindari pesan duplicate
          const alreadyExists = messages.some(
            (item) => item.id === message.id
          );

          if (alreadyExists) return;

          messages = [...messages, message];
        }
      )

      .subscribe((status) => {
        console.log("REALTIME STATUS:", status);
      });
  }

  // =========================
  // LOGOUT
  // =========================

  async function logout() {
    await supabase.auth.signOut();

    if (realtimeChannel) {
      await supabase.removeChannel(realtimeChannel);
    }

    await goto("/");
  }

  // =========================
  // START APP
  // =========================

  if (browser) {
    loadUser();
  }
</script>

<svelte:head>
  <title>ChatKita</title>
</svelte:head>

{#if loading}

  <div class="loading-screen">
    <div class="loading-box">
      <div class="spinner"></div>

      <h2>Memuat ChatKita...</h2>

      <p>
        Tunggu sebentar...
      </p>
    </div>
  </div>

{:else if errorMessage}

  <div class="error-screen">

    <div class="error-box">

      <h2>❌ Terjadi Error</h2>

      <p>{errorMessage}</p>

      <button onclick={() => location.reload()}>
        Coba Lagi
      </button>

    </div>

  </div>

{:else}

  <div class="app">

    <!-- SIDEBAR -->

    <aside class="sidebar">

      <div class="logo">
        💬
        <strong>ChatKita</strong>
      </div>

      <div class="account">

        <div class="avatar my-avatar">
          {user?.email?.charAt(0).toUpperCase()}
        </div>

        <div class="account-info">

          <strong>
            Akun Saya
          </strong>

          <small>
            {user?.email}
          </small>

        </div>

      </div>

      <div class="user-list">

        <h3>
          Pengguna
        </h3>

        {#if users.length === 0}

          <div class="empty">

            <p>
              Belum ada pengguna lain.
            </p>

            <small>
              Buat akun kedua untuk mencoba chat.
            </small>

          </div>

        {:else}

          {#each users as person}

            <button
              class="user-item"
              class:active={selectedUser?.id === person.id}
              onclick={() => openChat(person)}
            >

              <div class="avatar">

                {(
                  person.username ||
                  person.display_name ||
                  "U"
                )
                  .charAt(0)
                  .toUpperCase()}

              </div>

              <div class="user-info">

                <strong>
                  {person.display_name ||
                    person.username ||
                    "User"}
                </strong>

                <small>
                  @{person.username || "user"}
                </small>

              </div>

            </button>

          {/each}

        {/if}

      </div>

      <button
        class="logout"
        onclick={logout}
      >
        Keluar
      </button>

    </aside>

    <!-- CHAT AREA -->

    <main class="chat">

      {#if selectedUser}

        <!-- CHAT HEADER -->

        <div class="chat-header">

          <div class="avatar">

            {(
              selectedUser.username ||
              selectedUser.display_name ||
              "U"
            )
              .charAt(0)
              .toUpperCase()}

          </div>

          <div>

            <strong>
              {selectedUser.display_name ||
                selectedUser.username ||
                "User"}
            </strong>

            <p>
              @{selectedUser.username || "user"}
            </p>

          </div>

        </div>

        <!-- MESSAGES -->

        <div class="messages">

          {#if messages.length === 0}

            <div class="no-message">

              <div class="empty-icon">
                💬
              </div>

              <h3>
                Belum ada pesan
              </h3>

              <p>
                Kirim pesan pertama lu!
              </p>

            </div>

          {:else}

            {#each messages as message}

              <div
                class="message-wrapper"
                class:mine={message.sender_id === user.id}
              >

                <div class="message">

                  {message.content}

                </div>

                {#if message.created_at}

                  <small class="message-time">

                    {new Date(
                      message.created_at
                    ).toLocaleTimeString(
                      "id-ID",
                      {
                        hour: "2-digit",
                        minute: "2-digit"
                      }
                    )}

                  </small>

                {/if}

              </div>

            {/each}

          {/if}

        </div>

        <!-- MESSAGE INPUT -->

        <form
          class="message-form"
          onsubmit={(event) => {
            event.preventDefault();
            sendMessage();
          }}
        >

          <input
            type="text"
            placeholder="Ketik pesan..."
            bind:value={newMessage}
            autocomplete="off"
          />

          <button
            type="submit"
            disabled={!newMessage.trim()}
          >
            Kirim
          </button>

        </form>

      {:else}

        <!-- NO CHAT SELECTED -->

        <div class="select-chat">

          <div class="select-icon">
            💬
          </div>

          <h2>
            ChatKita
          </h2>

          <p>
            Pilih pengguna di sebelah kiri
            untuk mulai chatting.
          </p>

        </div>

      {/if}

    </main>

  </div>

{/if}

<style>

  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    font-family:
      Arial,
      Helvetica,
      sans-serif;

    background: #f5f5f5;
  }

  button,
  input {
    font-family: inherit;
  }

  /* =========================
     APP
  ========================= */

  .app {
    display: flex;
    width: 100%;
    height: 100vh;
    overflow: hidden;
  }

  /* =========================
     SIDEBAR
  ========================= */

  .sidebar {
    width: 320px;
    min-width: 320px;

    background: white;

    border-right:
      1px solid #ddd;

    display: flex;
    flex-direction: column;
  }

  .logo {
    padding: 22px 25px;

    font-size: 22px;

    border-bottom:
      1px solid #eee;
  }

  .account {
    display: flex;
    align-items: center;

    gap: 12px;

    padding: 18px;

    border-bottom:
      1px solid #eee;
  }

  .account-info {
    min-width: 0;
  }

  .account-info strong {
    display: block;
  }

  .account-info small {
    display: block;

    margin-top: 4px;

    color: #777;

    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  /* =========================
     USER LIST
  ========================= */

  .user-list {
    flex: 1;

    overflow-y: auto;

    padding: 15px;
  }

  .user-list h3 {
    margin:
      0 0 12px 0;
  }

  .user-item {
    width: 100%;

    display: flex;
    align-items: center;

    gap: 12px;

    padding: 12px;

    margin-bottom: 7px;

    background: transparent;

    border: none;

    border-radius: 10px;

    cursor: pointer;

    text-align: left;

    transition:
      background 0.15s;
  }

  .user-item:hover {
    background: #f2f4f7;
  }

  .user-item.active {
    background: #eef4ff;
  }

  .avatar {
    width: 42px;
    height: 42px;

    min-width: 42px;

    border-radius: 50%;

    background: #2563eb;

    color: white;

    display: flex;
    justify-content: center;
    align-items: center;

    font-weight: bold;
  }

  .my-avatar {
    background: #111827;
  }

  .user-info {
    min-width: 0;
  }

  .user-info strong {
    display: block;
  }

  .user-info small {
    display: block;

    color: #777;

    margin-top: 3px;
  }

  /* =========================
     LOGOUT
  ========================= */

  .logout {
    margin: 15px;

    padding: 12px;

    border: none;

    border-radius: 8px;

    background: #ef4444;

    color: white;

    cursor: pointer;

    font-size: 14px;
  }

  .logout:hover {
    background: #dc2626;
  }

  /* =========================
     CHAT
  ========================= */

  .chat {
    flex: 1;

    min-width: 0;

    display: flex;
    flex-direction: column;

    background: #f5f7fb;
  }

  /* =========================
     CHAT HEADER
  ========================= */

  .chat-header {
    display: flex;
    align-items: center;

    gap: 12px;

    background: white;

    padding:
      15px 25px;

    border-bottom:
      1px solid #ddd;
  }

  .chat-header strong {
    display: block;
  }

  .chat-header p {
    margin:
      4px 0 0;

    color: #777;

    font-size: 14px;
  }

  /* =========================
     MESSAGES
  ========================= */

  .messages {
    flex: 1;

    overflow-y: auto;

    padding: 25px;

    display: flex;
    flex-direction: column;

    gap: 8px;
  }

  .message-wrapper {
    display: flex;
    flex-direction: column;

    align-items: flex-start;

    max-width: 70%;
  }

  .message-wrapper.mine {
    align-self: flex-end;

    align-items: flex-end;
  }

  .message {
    padding:
      11px 15px;

    background: white;

    border-radius: 15px;

    box-shadow:
      0 2px 5px
      rgba(0, 0, 0, 0.05);

    word-break: break-word;
  }

  .message-wrapper.mine .message {
    background: #2563eb;

    color: white;
  }

  .message-time {
    margin-top: 3px;

    font-size: 10px;

    color: #999;
  }

  /* =========================
     MESSAGE FORM
  ========================= */

  .message-form {
    display: flex;

    gap: 10px;

    padding: 15px 20px;

    background: white;

    border-top:
      1px solid #ddd;
  }

  .message-form input {
    flex: 1;

    min-width: 0;

    padding: 13px;

    border:
      1px solid #ddd;

    border-radius: 10px;

    outline: none;

    font-size: 15px;
  }

  .message-form input:focus {
    border-color: #2563eb;
  }

  .message-form button {
    padding:
      13px 22px;

    border: none;

    border-radius: 10px;

    background: #2563eb;

    color: white;

    cursor: pointer;
  }

  .message-form button:hover {
    background: #1d4ed8;
  }

  .message-form button:disabled {
    background: #9ca3af;

    cursor: not-allowed;
  }

  /* =========================
     SELECT CHAT
  ========================= */

  .select-chat {
    margin: auto;

    text-align: center;

    color: #777;
  }

  .select-chat h2 {
    color: #222;

    margin-bottom: 8px;
  }

  .select-icon {
    font-size: 50px;
  }

  /* =========================
     EMPTY
  ========================= */

  .empty {
    padding: 15px;

    text-align: center;

    color: #777;
  }

  .empty p {
    margin-bottom: 5px;
  }

  .empty small {
    color: #999;
  }

  .no-message {
    margin: auto;

    text-align: center;

    color: #777;
  }

  .no-message h3 {
    color: #333;

    margin:
      10px 0 5px;
  }

  .no-message p {
    margin: 0;
  }

  .empty-icon {
    font-size: 45px;
  }

  /* =========================
     LOADING
  ========================= */

  .loading-screen {
    width: 100%;
    height: 100vh;

    display: flex;

    justify-content: center;
    align-items: center;

    background: #f5f7fb;
  }

  .loading-box {
    text-align: center;

    color: #555;
  }

  .loading-box h2 {
    color: #222;

    margin:
      15px 0 5px;
  }

  .loading-box p {
    margin: 0;

    color: #888;
  }

  .spinner {
    width: 40px;
    height: 40px;

    margin: auto;

    border:
      4px solid #ddd;

    border-top-color:
      #2563eb;

    border-radius: 50%;

    animation:
      spin 0.8s linear infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  /* =========================
     ERROR
  ========================= */

  .error-screen {
    width: 100%;
    height: 100vh;

    display: flex;

    justify-content: center;
    align-items: center;

    background: #f5f7fb;
  }

  .error-box {
    max-width: 500px;

    padding: 30px;

    background: white;

    border-radius: 12px;

    text-align: center;

    box-shadow:
      0 5px 20px
      rgba(0, 0, 0, 0.08);
  }

  .error-box h2 {
    margin-top: 0;

    color: #dc2626;
  }

  .error-box p {
    color: #555;

    word-break: break-word;
  }

  .error-box button {
    padding:
      11px 20px;

    border: none;

    border-radius: 8px;

    background: #2563eb;

    color: white;

    cursor: pointer;
  }

  /* =========================
     MOBILE
  ========================= */

  @media (max-width: 700px) {

    .sidebar {
      width: 250px;
      min-width: 250px;
    }

    .message-wrapper {
      max-width: 85%;
    }

    .messages {
      padding: 15px;
    }

    .message-form {
      padding: 10px;
    }

  }

</style>