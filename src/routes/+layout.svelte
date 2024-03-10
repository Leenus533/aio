<script>
  import "../app.pcss";
  import Search from "$lib/components/Search.svelte";
  import Button from "$lib/components/ui/button/button.svelte";
  import { page } from "$app/stores";

  // get current path if the page is settings then show home button else show settings button
  $: buttonLabel = $page.url.pathname === "/settings" ? "Home" : "Settings";
  $: buttonLink = $page.url.pathname === "/settings" ? "/" : "/settings";
</script>

<div>
  <div class="border-b">
    <div class="flex h-16 items-center px-4">
      <a href="/" class="text-lg font-bold">AIo</a>
      <div class="ml-auto flex items-center space-x-4">
        <Search />
        <Button href={buttonLink}>{buttonLabel}</Button>
      </div>
    </div>
  </div>

  <slot />
</div>
