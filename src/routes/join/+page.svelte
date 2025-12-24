<script>
  import { goto } from "$app/navigation";
  import { supabase } from "$lib/supabase";

  // steps: join flow ("code" -> "join_name") or create flow ("create")
  let steps = "code"; // "code" | "join_name" | "create"

  // join flow
  let join_code = "";
  let join_display_name = "";
  let verifiedHousehold = null;

  // create flow
  let household_name = "";
  let open_time = "07:00";
  let close_time = "20:00";
  let create_display_name = "";

  // timezone (sent to DB; not shown in UI)
  let timezone = "";

  let globalError = "";
  let saving = false;

  function getTimezone() {
    try {
      return Intl.DateTimeFormat().resolvedOptions().timeZone || "UTC";
    } catch {
      return "UTC";
    }
  }

  // initialize timezone on load (client-side)
  timezone = getTimezone();

  async function verifyCode() {
    globalError = "";

    if (join_code.length !== 6) {
      globalError = "Please enter a valid 6-character household code.";
      return;
    }

    const { data: household, error } = await supabase
      .from("households")
      .select("*")
      .eq("code", join_code.toUpperCase())
      .maybeSingle();

    if (!household || error) {
      console.error("Error verifying code:", error);
      globalError = "Invalid household code. Please try again.";
      return;
    }

    // ensure we have a user id for membership insert
    await supabase.auth.signInAnonymously();

    verifiedHousehold = household;
    steps = "join_name";
  }

  async function joinHousehold() {
    globalError = "";

    if (!join_display_name.trim()) {
      globalError = "Please enter your name.";
      return;
    }

    const {
      data: { user },
      error: userErr,
    } = await supabase.auth.getUser();

    if (!user || userErr) {
      globalError = "Could not get user session. Please try again.";
      return;
    }

    const { error } = await supabase.from("household_members").insert({
      household_id: verifiedHousehold.id,
      user_id: user.id,
      display_name: join_display_name.trim(),
    });

    if (error) {
      console.error("Error joining household:", error);
      globalError = "Couldn't join household. Please try again.";
      return;
    }

    goto("/");
  }

  function openCreateModal() {
    globalError = "";
    timezone = getTimezone();
    steps = "create";
  }

  async function createHousehold() {
    globalError = "";

    if (!household_name.trim()) {
      globalError = "Please enter a household name.";
      return;
    }
    if (!create_display_name.trim()) {
      globalError = "Please enter your name.";
      return;
    }
    if (!open_time || !close_time) {
      globalError = "Please choose open and close times.";
      return;
    }
    if (open_time === close_time) {
      globalError = "Open and close times can't be the same.";
      return;
    }

    saving = true;

    try {
      // ensure we have a user id
      await supabase.auth.signInAnonymously();

      const {
        data: { user },
        error: userErr,
      } = await supabase.auth.getUser();

      if (!user || userErr) {
        globalError = "Could not get user session. Please try again.";
        return;
      }

      // 1) create household
      const { data: household, error: householdErr } = await supabase
        .from("households")
        .insert({
          name: household_name.trim(),
          open_time, // "HH:MM"
          close_time, // "HH:MM"
          timezone, // TEXT column (IANA tz string)
        })
        .select("*")
        .single();

      if (householdErr || !household) {
        console.error("Error creating household:", householdErr);
        globalError = "Couldn't create household. Please try again.";
        return;
      }

      // 2) add creator as member
      const { error: memberErr } = await supabase
        .from("household_members")
        .insert({
          household_id: household.id,
          user_id: user.id,
          display_name: create_display_name.trim(),
        });

      if (memberErr) {
        console.error("Error adding member:", memberErr);
        globalError = "Household created, but couldn't add you as a member.";
        return;
      }

      goto("/");
    } finally {
      saving = false;
    }
  }
</script>

<div class="container">
  <div class="header">
    <h1 class="logo">Cup o' Cheer</h1>
    <p class="subtitle">Join your household's coffee shop</p>
  </div>

  <div class="join-card">
    {#if globalError}
      <p class="error-container">{globalError}</p>
    {/if}

    <div class="input-group">
      <label for="code">Household Code</label>
      <input
        bind:value={join_code}
        maxlength="6"
        type="text"
        id="code"
        placeholder="Enter code"
        style="text-transform: uppercase;"
      />
    </div>

    <button on:click={verifyCode} class="join-btn">Join</button>

    <div class="divider">
      <span>or</span>
    </div>

    <button class="create-btn" on:click={openCreateModal}
      >Start New Household</button
    >
  </div>
</div>

{#if steps === "join_name"}
  <div class="modal" id="joinModal">
    <div class="modal-content">
      <div class="form-group">
        <label for="yourNameJoin">Your Name</label>
        <input
          bind:value={join_display_name}
          type="text"
          id="yourNameJoin"
          placeholder="What should we call you?"
        />
      </div>

      <button on:click={joinHousehold} class="submit-btn">Join</button>
    </div>
  </div>
{/if}

{#if steps === "create"}
  <div class="modal" id="createModal">
    <div class="modal-content">
      <div class="form-group">
        <label for="householdName">Household Name</label>
        <input
          bind:value={household_name}
          type="text"
          id="householdName"
          placeholder="e.g. The Smiths"
        />
      </div>

      <div class="form-group">
        <label for="openTime">Open Time</label>
        <input bind:value={open_time} type="time" id="openTime" />
      </div>

      <div class="form-group">
        <label for="closeTime">Close Time</label>
        <input bind:value={close_time} type="time" id="closeTime" />
      </div>

      <div class="form-group">
        <label for="yourNameCreate">Your Name</label>
        <input
          bind:value={create_display_name}
          type="text"
          id="yourNameCreate"
          placeholder="What should we call you?"
        />
      </div>

      <button disabled={saving} on:click={createHousehold} class="submit-btn">
        {saving ? "Creating..." : "Create Household"}
      </button>
    </div>
  </div>
{/if}

<style>
  .container {
    max-width: 450px;
    width: 100%;
  }

  .header {
    text-align: center;
    margin-bottom: 3rem;
  }

  .logo {
    font-size: 2.5rem;
    font-weight: 700;
    color: #6b4423;
    letter-spacing: -0.5px;
    margin-bottom: 0.5rem;
  }

  .subtitle {
    color: #8d6e63;
    font-size: 1rem;
  }

  .error-container {
    background-color: #ffdddd;
    border: 1px solid #ff5c5c;
    padding: 1rem;
    margin: 1rem 0;
    border-radius: 5px;
    color: #a70000;
  }

  .join-card {
    background: white;
    padding: 2.5rem;
    border-radius: 16px;
    box-shadow: 0 2px 8px rgba(107, 68, 35, 0.08);
  }

  .input-group {
    margin-bottom: 1rem;
  }

  .input-group label {
    display: block;
    color: #5d4037;
    margin-bottom: 0.5rem;
    font-weight: 500;
  }

  input[type="text"] {
    width: 100%;
    padding: 0.875rem;
    border: 2px solid #e0d5c7;
    border-radius: 8px;
    font-size: 1rem;
    font-family: inherit;
    transition: border-color 0.2s ease;
    text-align: center;
    letter-spacing: 2px;
    font-weight: 600;
  }

  input[type="text"]:focus {
    outline: none;
    border-color: #d4a574;
  }

  input[type="text"]::placeholder {
    text-transform: none;
    letter-spacing: normal;
    font-weight: normal;
  }

  input[type="time"] {
    width: 100%;
    padding: 0.875rem;
    border: 2px solid #e0d5c7;
    border-radius: 8px;
    font-size: 1rem;
    font-family: inherit;
    transition: border-color 0.2s ease;
  }

  input[type="time"]:focus {
    outline: none;
    border-color: #d4a574;
  }

  .join-btn {
    width: 100%;
    background: #d4a574;
    color: white;
    border: none;
    padding: 1rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
    margin-bottom: 1rem;
  }

  .join-btn:hover {
    background: #c9955f;
    transform: translateY(-1px);
  }

  .divider {
    display: flex;
    align-items: center;
    text-align: center;
    margin: 1.5rem 0;
    color: #8d6e63;
    font-size: 0.9rem;
  }

  .divider::before,
  .divider::after {
    content: "";
    flex: 1;
    border-bottom: 1px solid #e0d5c7;
  }

  .divider span {
    padding: 0 1rem;
  }

  .create-btn {
    width: 100%;
    background: white;
    color: #6b4423;
    border: 2px solid #d4a574;
    padding: 1rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .create-btn:hover {
    background: #faf6f1;
    border-color: #c9955f;
  }

  .modal {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, 0.5);
    z-index: 1000;
    padding: 1rem;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .modal-content {
    background: white;
    padding: 2rem;
    border-radius: 16px;
    max-width: 500px;
    width: 100%;
    max-height: 90vh;
    overflow-y: auto;
  }

  .form-group {
    margin-bottom: 1.5rem;
  }

  .form-group label {
    display: block;
    color: #5d4037;
    margin-bottom: 0.5rem;
    font-weight: 600;
  }

  .form-group input {
    width: 100%;
    padding: 0.875rem;
    border: 2px solid #e0d5c7;
    border-radius: 8px;
    font-size: 1rem;
    font-family: inherit;
    transition: border-color 0.2s ease;
  }

  .form-group input:focus {
    outline: none;
    border-color: #d4a574;
  }

  .submit-btn {
    width: 100%;
    background: #d4a574;
    color: white;
    border: none;
    padding: 1rem;
    border-radius: 8px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.2s ease;
  }

  .submit-btn:hover:enabled {
    background: #c9955f;
    transform: translateY(-1px);
  }

  .submit-btn:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }
</style>
