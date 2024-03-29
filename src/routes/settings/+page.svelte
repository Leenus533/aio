<script lang="ts">
  import { onMount } from "svelte";
  import { Button } from "$lib/components/ui/button";
  import { env } from "$env/dynamic/public";
  import SocialIcons from "@rodneylab/svelte-social-icons";
  //@ts-ignore
  import { getCookie, setCookie, deleteCookie } from "svelte-cookie";

  const redirect_uri = "http://localhost:5173/settings";

  let gmailAuthenticated = false;
  let discordAuthenticated = false;

  onMount(() => {
    const urlParamsWithHash = new URLSearchParams(
      window.location.hash.slice(1)
    );
    const access_token = urlParamsWithHash.get("access_token");
    const state = urlParamsWithHash.get("state");
    if (access_token && state === "gmail") {
      setCookie("gmail_access_token", access_token, { path: "/" });
      gmailAuthenticated = true;
      window.history.replaceState(null, "", window.location.pathname);
    }

    const urlParams = new URLSearchParams(window.location.search);
    const code = urlParams.get("code");

    if (code) {
      exchangeDiscordCodeForAccessToken(code);
      window.history.replaceState(null, "", window.location.pathname);
    }

    gmailAuthenticated = getCookie("gmail_access_token") != "";
    discordAuthenticated = getCookie("discord_access_token") != "";
  });

  const handleGmailConnectClick = () => {
    const clientId = env.PUBLIC_GOOGLE_CLIENT_ID;
    const scope = "https://www.googleapis.com/auth/gmail.readonly";
    const authUrl = `https://accounts.google.com/o/oauth2/auth?client_id=${clientId}&redirect_uri=${encodeURIComponent(redirect_uri)}&response_type=token&scope=${scope}&state=gmail`;
    window.location.href = authUrl;
  };

  const handleDiscordConnectClick = () => {
    const clientId = env.PUBLIC_DISCORD_CLIENT_ID;
    const scope = "bot";
    const authUrl = `https://discord.com/api/oauth2/authorize?client_id=${clientId}&redirect_uri=${encodeURIComponent(redirect_uri)}&response_type=code&scope=${scope}`;
    window.location.href = authUrl;
  };

  const exchangeDiscordCodeForAccessToken = async (code: any) => {
    try {
      const clientId = env.PUBLIC_DISCORD_CLIENT_ID;
      const clientSecret = env.PUBLIC_DISCORD_SECRET;
      const response = await fetch("https://discord.com/api/oauth2/token", {
        method: "POST",
        headers: {
          "Content-Type": "application/x-www-form-urlencoded",
        },
        body: `client_id=${clientId}&client_secret=${clientSecret}&grant_type=authorization_code&code=${code}&redirect_uri=${redirect_uri}`,
      });
      const data = await response.json();
      const accessToken = data.access_token;
      setCookie("discord_access_token", accessToken, { path: "/" });
      discordAuthenticated = true;
    } catch (error) {
      console.error("Error exchanging code for access token:", error);
    }
  };
</script>

<main class="container mx-auto p-4 flex items-center justify-center">
  <div>
    <h1 class="text-2xl font-bold mb-4">Settings</h1>
    <div class="flex flex-col items-center justify-center">
      <div class="flex items-center">
        <SocialIcons network="google" width={24} height={24} />
        <h2 class="text-xl font-bold mb-2 text-center">Gmail</h2>
      </div>
      {#if gmailAuthenticated}
        <Button
          class="mx-auto"
          on:click={() => {
            deleteCookie("gmail_access_token"), (gmailAuthenticated = false);
          }}
        >
          Disconnect
        </Button>
      {:else}
        <Button on:click={handleGmailConnectClick}>Connect to Gmail</Button>
      {/if}

      <div class="flex items-center">
        <SocialIcons network="discord" width={24} height={24} />
        <h2 class="text-xl font-bold mb-2 text-center">Discord</h2>
      </div>
      {#if discordAuthenticated}
        <Button
          class="mx-auto"
          on:click={() => {
            deleteCookie("discord_access_token");
            discordAuthenticated = false;
          }}
        >
          Disconnect
        </Button>
      {:else}
        <Button disabled on:click={handleDiscordConnectClick}
          >Connect to Discord</Button
        >
      {/if}

      <div class="flex items-center">
        <SocialIcons network="facebook" width={24} height={24} />
        <h2 class="text-xl font-bold mb-2 text-center">Facebook</h2>
      </div>
      <Button disabled class="mx-auto">Connect to Facebook</Button>
      <div class="flex items-center">
        <SocialIcons network="twitter" width={24} height={24} />
        <h2 class="text-xl font-bold mb-2 text-center">x</h2>
      </div>
      <Button disabled class="mx-auto">Connect to x</Button>
      <div class="flex items-center">
        <SocialIcons network="reddit" width={24} height={24} />
        <h2 class="text-xl font-bold mb-2 text-center">Reddit</h2>
      </div>
      <Button disabled>Connect to Reddit</Button>
    </div>
  </div>
</main>
