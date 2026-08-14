<script>
  import { goto } from "$app/navigation";
  import { supabase } from "$lib/supabase";

  let username = $state("");
  let errorMessage = $state("");
  let loading = $state(false);

  function generateNumber() {
    return Math.floor(100000 + Math.random() * 900000).toString();
  }

  async function buatNomorUnik() {
    for (let i = 0; i < 10; i++) {
      const nomor = generateNumber();

      const { data, error } = await supabase
        .from("profiles")
        .select("nomor")
        .eq("nomor", nomor)
        .maybeSingle();

      if (error) {
        throw error;
      }

      if (!data) {
        return nomor;
      }
    }

    throw new Error("Gagal membuat nomor unik. Coba lagi.");
  }

  async function masuk() {
    errorMessage = "";

    const usernameBersih = username.trim();

    if (!usernameBersih) {
      errorMessage = "Username wajib diisi.";
      return;
    }

    loading = true;

    try {
      const { data: userLama, error: errorCari } = await supabase
        .from("profiles")
        .select("*")
        .eq("username", usernameBersih)
        .maybeSingle();

      if (errorCari) {
        throw errorCari;
      }

      let nomorUser;

      if (userLama) {
        nomorUser = userLama.nomor;
      } else {
        nomorUser = await buatNomorUnik();

        const { data: profileBaru, error: errorInsert } = await supabase
          .from("profiles")
          .insert({
            id: crypto.randomUUID(),
            username: usernameBersih,
            display_name: usernameBersih,
            nomor: nomorUser
          })
          .select()
          .single();

        if (errorInsert) {
          throw errorInsert;
        }

        nomorUser = profileBaru.nomor;
      }

      localStorage.setItem("username", usernameBersih);
      localStorage.setItem("nomorUser", nomorUser);

      await goto("/chat");
    } catch (error) {
      console.error("ERROR MASUK:", error);

      errorMessage =
        error instanceof Error
          ? error.message
          : "Terjadi kesalahan.";
    } finally {
      loading = false;
    }
  }
</script>

<svelte:head>
  <title>ChatKita</title>
</svelte:head>

<div class="login-container">
  <div class="login-box">
    <h1>ChatKita</h1>

    <p class="subtitle">
      Masukkan username untuk mulai chat
    </p>

    <div class="input-group">
      <label for="username">Username</label>

      <input
        id="username"
        type="text"
        placeholder="Masukkan username"
        bind:value={username}
        autocomplete="username"
        disabled={loading}
        onkeydown={(event) => {
          if (event.key === "Enter") {
            masuk();
          }
        }}
      />
    </div>

    {#if errorMessage}
      <p class="error">{errorMessage}</p>
    {/if}

    <button
      type="button"
      onclick={masuk}
      disabled={loading}
    >
      {loading ? "Masuk..." : "Masuk"}
    </button>
  </div>
</div>

<style>
  .login-container {
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    background: #f5f5f5;
    padding: 20px;
  }

  .login-box {
    width: 100%;
    max-width: 400px;
    background: white;
    padding: 30px;
    border-radius: 12px;
    box-shadow: 0 5px 20px rgba(0, 0, 0, 0.1);
  }

  h1 {
    text-align: center;
    margin: 0;
    font-size: 32px;
  }

  .subtitle {
    text-align: center;
    color: #666;
    margin-bottom: 25px;
  }

  .input-group {
    margin-bottom: 18px;
  }

  label {
    display: block;
    margin-bottom: 7px;
    font-weight: 600;
  }

  input {
    width: 100%;
    box-sizing: border-box;
    padding: 12px;
    border: 1px solid #ccc;
    border-radius: 8px;
    font-size: 15px;
  }

  input:focus {
    outline: none;
    border-color: #555;
  }

  button {
    width: 100%;
    padding: 12px;
    border: none;
    border-radius: 8px;
    background: #111;
    color: white;
    font-size: 16px;
    cursor: pointer;
  }

  button:disabled,
  input:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .error {
    color: #dc2626;
    margin-bottom: 15px;
    font-size: 14px;
  }
</style>