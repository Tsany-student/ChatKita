<script>
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabase";

  /**
   * @typedef {{
   *   id: string,
   *   username: string,
   *   display_name: string | null,
   *   avatar_url: string | null,
   *   bio: string | null,
   *   created_at: string,
   *   nomor: string
   * }} Profile
   *
   * @typedef {{
   *   id: string,
   *   sender_nomor: string,
   *   receiver_nomor: string,
   *   message: string,
   *   created_at: string
   * }} Message
   */

  let username = $state("");
  let nomorUser = $state("");
  let nomorCari = $state("");
  let pesan = $state("");

  /** @type {Profile | null} */
  let selectedUser = $state(null);

  /** @type {Profile[]} */
  let chats = $state([]);

  /** @type {Message[]} */
  let messages = $state([]);

  let loadingUser = $state(true);
  let searching = $state(false);
  let sending = $state(false);
  let errorMessage = $state("");

  onMount(() => {
    /** @type {any} */
    let channel = null;

    async function mulai() {
      username = localStorage.getItem("username") || "";
      nomorUser = localStorage.getItem("nomorUser") || "";

      loadingUser = false;

      await loadChats();

      channel = supabase
        .channel("messages-realtime")
        .on(
          "postgres_changes",
          {
            event: "INSERT",
            schema: "public",
            table: "messages"
          },
          (payload) => {
            /** @type {Message} */
            const message = /** @type {Message} */ (payload.new);

            const berhubunganDenganSaya =
              message.sender_nomor === nomorUser ||
              message.receiver_nomor === nomorUser;

            if (!berhubunganDenganSaya) return;

            if (
              selectedUser &&
              (
                (
                  message.sender_nomor === nomorUser &&
                  message.receiver_nomor === selectedUser.nomor
                ) ||
                (
                  message.sender_nomor === selectedUser.nomor &&
                  message.receiver_nomor === nomorUser
                )
              )
            ) {
              const sudahAda = messages.some(
                (item) => item.id === message.id
              );

              if (!sudahAda) {
                messages = [...messages, message];
              }
            }

            loadChats();
          }
        )
        .subscribe();
    }

    mulai();

    return () => {
      if (channel) {
        supabase.removeChannel(channel);
      }
    };
  });

  async function loadChats() {
    if (!nomorUser) return;

    const { data, error } = await supabase
      .from("messages")
      .select("*")
      .or(
        `sender_nomor.eq.${nomorUser},receiver_nomor.eq.${nomorUser}`
      )
      .order("created_at", { ascending: false });

    if (error || !data) {
      console.error("ERROR LOAD CHATS:", error);
      return;
    }

    const nomorSudahAda = new Set();

    for (const message of data) {
      const nomorTeman =
        message.sender_nomor === nomorUser
          ? message.receiver_nomor
          : message.sender_nomor;

      nomorSudahAda.add(nomorTeman);
    }

    const nomorArray = [...nomorSudahAda];

    if (nomorArray.length === 0) {
      chats = [];
      return;
    }

    const { data: users, error: errorUsers } = await supabase
      .from("profiles")
      .select("*")
      .in("nomor", nomorArray);

    if (errorUsers) {
      console.error("ERROR LOAD USERS:", errorUsers);
      return;
    }

    /** @type {Profile[]} */
    chats = users || [];
  }

  async function cariUser() {
    errorMessage = "";

    const nomor = nomorCari.trim();

    if (!nomor) {
      errorMessage = "Masukkan nomor ID teman.";
      return;
    }

    if (nomor === nomorUser) {
      errorMessage = "Itu nomor ID kamu sendiri 😭";
      return;
    }

    searching = true;

    try {
      const { data, error } = await supabase
        .from("profiles")
        .select("*")
        .eq("nomor", nomor)
        .maybeSingle();

      if (error) throw error;

      if (!data) {
        errorMessage = "User dengan nomor tersebut tidak ditemukan.";
        return;
      }

      /** @type {Profile} */
      const user = data;

      selectedUser = user;

      const sudahAda = chats.some(
        (chat) => chat.nomor === user.nomor
      );

      if (!sudahAda) {
        chats = [user, ...chats];
      }

      nomorCari = "";

      await loadMessages(user.nomor);
    } catch (error) {
      console.error("ERROR CARI USER:", error);

      errorMessage =
        error instanceof Error
          ? error.message
          : "Gagal mencari user.";
    } finally {
      searching = false;
    }
  }

  /**
   * @param {Profile} chat
   */
  async function pilihChat(chat) {
    selectedUser = chat;
    errorMessage = "";

    await loadMessages(chat.nomor);
  }

  /**
   * @param {string} nomorTeman
   */
  async function loadMessages(nomorTeman) {
    if (!nomorUser || !nomorTeman) return;

    const { data, error } = await supabase
      .from("messages")
      .select("*")
      .or(
        `and(sender_nomor.eq.${nomorUser},receiver_nomor.eq.${nomorTeman}),and(sender_nomor.eq.${nomorTeman},receiver_nomor.eq.${nomorUser})`
      )
      .order("created_at", { ascending: true });

    if (error) {
      console.error("ERROR LOAD MESSAGE:", error);
      errorMessage = error.message;
      return;
    }

    /** @type {Message[]} */
    messages = data || [];
  }

  async function kirimPesan() {
    if (!pesan.trim()) return;
    if (!selectedUser) return;

    sending = true;
    errorMessage = "";

    try {
      const isiPesan = pesan.trim();

      const { data, error } = await supabase
        .from("messages")
        .insert({
          sender_nomor: nomorUser,
          receiver_nomor: selectedUser.nomor,
          message: isiPesan
        })
        .select()
        .single();

      if (error) throw error;

      /** @type {Message} */
      const message = data;

      const sudahAda = messages.some(
        (item) => item.id === message.id
      );

      if (!sudahAda) {
        messages = [...messages, message];
      }

      pesan = "";

      const user = selectedUser;

      const sudahAdaChat = chats.some(
        (chat) => chat.nomor === user.nomor
      );

      if (!sudahAdaChat) {
        chats = [user, ...chats];
      }
    } catch (error) {
      console.error("ERROR KIRIM:", error);

      errorMessage =
        error instanceof Error
          ? error.message
          : "Pesan gagal dikirim.";
    } finally {
      sending = false;
    }
  }

  /**
   * @param {string} waktu
   */
  function formatWaktu(waktu) {
    return new Date(waktu).toLocaleTimeString("id-ID", {
      hour: "2-digit",
      minute: "2-digit"
    });
  }
</script>

<svelte:head>
  <title>ChatKita</title>
</svelte:head>

<div class="app">
  <aside class="sidebar">
    <div class="logo">
      <h1>ChatKita</h1>
    </div>

    <div class="search-box">
      <input
        type="text"
        placeholder="Masukkan nomor ID..."
        bind:value={nomorCari}
        disabled={searching}
        onkeydown={(event) => {
          if (event.key === "Enter") {
            cariUser();
          }
        }}
      />

      <button
        onclick={cariUser}
        disabled={searching}
      >
        {searching ? "..." : "Cari"}
      </button>
    </div>

    {#if errorMessage}
      <p class="error">{errorMessage}</p>
    {/if}

    <div class="chat-list">
      {#if chats.length === 0}
        <p class="empty">
          Cari teman menggunakan nomor ID
        </p>
      {/if}

      {#each chats as chat}
        <button
          class:selected={selectedUser?.nomor === chat.nomor}
          class="chat-user"
          onclick={() => pilihChat(chat)}
        >
          <div class="avatar">
            {(chat.username || "U").charAt(0).toUpperCase()}
          </div>

          <div class="chat-user-info">
            <b>{chat.display_name || chat.username}</b>
            <span>ID: {chat.nomor}</span>
          </div>
        </button>
      {/each}
    </div>

    <div class="profile">
      <div class="profile-avatar">
        {username
          ? username.charAt(0).toUpperCase()
          : "U"}
      </div>

      <div class="profile-info">
        <b>{username || "User"}</b>
        <span>ID: {nomorUser}</span>
      </div>
    </div>
  </aside>

  <main class="chat-area">
    {#if loadingUser}

      <div class="no-chat">
        <p>Memuat...</p>
      </div>

    {:else if selectedUser}

      <div class="chat-header">
        <div class="avatar">
          {(selectedUser.username || "U").charAt(0).toUpperCase()}
        </div>

        <div>
          <b>
            {selectedUser.display_name || selectedUser.username}
          </b>

          <p>ID: {selectedUser.nomor}</p>
        </div>
      </div>

      <div class="messages">
        {#if messages.length === 0}
          <div class="welcome-message">
            <p>
              Mulai percakapan dengan
              {selectedUser.display_name || selectedUser.username}
            </p>
          </div>
        {/if}

        {#each messages as item}
          <div
            class:mine={item.sender_nomor === nomorUser}
            class="message-row"
          >
            <div class="message-bubble">
              <p>{item.message}</p>

              <span>
                {formatWaktu(item.created_at)}
              </span>
            </div>
          </div>
        {/each}
      </div>

      <div class="message-input">
        <input
          type="text"
          placeholder="Tulis pesan..."
          bind:value={pesan}
          disabled={sending}
          onkeydown={(event) => {
            if (event.key === "Enter") {
              kirimPesan();
            }
          }}
        />

        <button
          onclick={kirimPesan}
          disabled={sending}
        >
          {sending ? "..." : "Kirim"}
        </button>
      </div>

    {:else}

      <div class="no-chat">
        <h2>Selamat datang di ChatKita 👋</h2>

        <p>
          Masukkan nomor ID teman di sebelah kiri
          untuk mulai chat.
        </p>
      </div>

    {/if}
  </main>
</div>

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    font-family: Arial, sans-serif;
  }

  .app {
    display: flex;
    height: 100vh;
    background: #f5f5f5;
  }

  .sidebar {
    width: 340px;
    background: white;
    border-right: 1px solid #ddd;
    display: flex;
    flex-direction: column;
  }

  .logo {
    padding: 20px;
    border-bottom: 1px solid #eee;
  }

  .logo h1 {
    margin: 0;
    font-size: 24px;
  }

  .search-box {
    display: flex;
    gap: 8px;
    padding: 15px;
  }

  .search-box input {
    width: 100%;
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 8px;
  }

  .search-box button {
    padding: 10px 14px;
    border: none;
    border-radius: 8px;
    background: #111;
    color: white;
    cursor: pointer;
  }

  .error {
    margin: 0 15px 10px;
    color: #dc2626;
    font-size: 13px;
  }

  .chat-list {
    flex: 1;
    overflow-y: auto;
    border-top: 1px solid #eee;
  }

  .empty {
    text-align: center;
    color: #777;
    padding: 30px 20px;
    font-size: 14px;
  }

  .chat-user {
    width: 100%;
    border: none;
    background: white;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 15px;
    cursor: pointer;
    text-align: left;
    border-bottom: 1px solid #eee;
  }

  .chat-user:hover,
  .chat-user.selected {
    background: #f2f2f2;
  }

  .avatar,
  .profile-avatar {
    width: 42px;
    height: 42px;
    border-radius: 50%;
    background: #111;
    color: white;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    flex-shrink: 0;
  }

  .chat-user-info,
  .profile-info {
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  .chat-user-info span,
  .profile-info span {
    font-size: 12px;
    color: #777;
  }

  .profile {
    border-top: 1px solid #ddd;
    padding: 15px;
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .chat-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-width: 0;
  }

  .no-chat {
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    text-align: center;
    color: #666;
    padding: 20px;
  }

  .no-chat h2 {
    color: #111;
    margin-bottom: 5px;
  }

  .chat-header {
    height: 73px;
    background: white;
    border-bottom: 1px solid #ddd;
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 15px 25px;
  }

  .chat-header p {
    margin: 3px 0 0;
    font-size: 12px;
    color: #777;
  }

  .messages {
    flex: 1;
    padding: 20px;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .welcome-message {
    text-align: center;
    color: #777;
    margin-top: 30px;
  }

  .message-row {
    display: flex;
    justify-content: flex-start;
  }

  .message-row.mine {
    justify-content: flex-end;
  }

  .message-bubble {
    max-width: 70%;
    background: white;
    padding: 10px 14px;
    border-radius: 12px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.08);
  }

  .mine .message-bubble {
    background: #111;
    color: white;
  }

  .message-bubble p {
    margin: 0;
    word-break: break-word;
  }

  .message-bubble span {
    display: block;
    margin-top: 5px;
    font-size: 10px;
    opacity: 0.65;
  }

  .message-input {
    display: flex;
    gap: 10px;
    padding: 15px;
    background: white;
    border-top: 1px solid #ddd;
  }

  .message-input input {
    flex: 1;
    padding: 12px 15px;
    border: 1px solid #ccc;
    border-radius: 8px;
    font-size: 15px;
  }

  .message-input button {
    padding: 12px 20px;
    border: none;
    border-radius: 8px;
    background: #111;
    color: white;
    cursor: pointer;
  }

  button:disabled,
  input:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  @media (max-width: 700px) {
    .sidebar {
      width: 280px;
    }
  }
</style>