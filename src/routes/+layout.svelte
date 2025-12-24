<script>
  import { supabase } from "$lib/supabase";
  import { onMount } from "svelte";
  import { page } from "$app/state";
  import { goto } from "$app/navigation";
  import "../app.css";

  let { children } = $props();

  let user = null;

  onMount(async () => {
    const {
      data: { session },
    } = await supabase.auth.getSession();
    user = session?.user ?? null;

    const currentPath = page.url.pathname;
    console.log("Current path:", currentPath);

    if (!user) {
      if (currentPath != "/join") {
        goto("/join");
      }
    } else {
      if (currentPath == "/join") {
        goto("/");
      }
    }
  });
</script>

<svelte:head>
  <title>{page.data.title ?? ""} - Cup O' Cheer</title>
  <link
    rel="icon"
    href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>☕</text></svg>"
  />
</svelte:head>

{@render children()}
