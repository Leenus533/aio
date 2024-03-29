<script lang="ts">
  import { onMount } from "svelte";
  import { Button } from "$lib/components/ui/button";
  import { env } from "$env/dynamic/public";
  import * as RadioGroup from "$lib/components/ui/radio-group";
  import * as Card from "$lib/components/ui/card";
  import { Label } from "$lib/components/ui/label";
  //@ts-ignore
  import { getCookie } from "svelte-cookie";
  import {
    Dialog,
    DialogContent,
    DialogTitle,
    DialogTrigger,
  } from "$lib/components/ui/dialog";

  let content: any[] = [];
  let gmailAuthenticated = false;
  let discordAuthenticated = false;
  let selectedCategory = "all";
  let summarizeDialog = false;
  let summaryLoading = false;
  let summary = "";

  onMount(() => {
    // Check if the user is authenticated
    gmailAuthenticated = getCookie("gmail_access_token") != "";
    discordAuthenticated = getCookie("discord_access_token") != "";

    if (gmailAuthenticated) {
      fetchEmails();
    }
  });

  const fetchEmails = async () => {
    const accessToken = getCookie("gmail_access_token");
    try {
      const response = await fetch(
        "https://gmail.googleapis.com/gmail/v1/users/me/messages?maxResults=20",
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
          },
        }
      );
      const data = await response.json();
      const messageIds = data.messages.map((message: any) => message.id);
      const emailPromises = messageIds.map((messageId: string) =>
        fetchEmailContent(messageId, accessToken)
      );
      const emails = await Promise.all(emailPromises);
      content = [...content, ...emails];
      console.log(`Latest ${emails.length} emails:`, emails);
    } catch (error) {
      console.error("Error fetching emails:", error);
    }
  };

  const fetchEmailContent = async (messageId: string, accessToken: any) => {
    try {
      const response = await fetch(
        `https://gmail.googleapis.com/gmail/v1/users/me/messages/${messageId}`,
        {
          headers: {
            Authorization: `Bearer ${accessToken}`,
          },
        }
      );
      const data = await response.json();
      const subject = data.payload.headers.find(
        (header: { name: string }) => header.name === "Subject"
      )?.value;
      const snippet = data.snippet;
      const category = await categorizeEmail(subject, snippet);
      return {
        id: messageId,
        subject,
        snippet,
        category,
        platform: "Gmail",
      };
    } catch (error) {
      console.error("Error fetching email content:", error);
      return null;
    }
  };

  const categorizeEmail = async (subject: any, snippet: any) => {
    const apiKey = env.PUBLIC_OPENAI_API_KEY;
    const prompt = `Categorize the following email based on its subject and snippet into one of four categories: "Work", "Fun/Social", "News/Info", "Spam/Advertisement".

Subject: ${subject}
Snippet: ${snippet}

Category:`;

    try {
      const response = await fetch(
        "https://api.openai.com/v1/chat/completions",
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${apiKey}`,
          },
          body: JSON.stringify({
            model: "gpt-3.5-turbo",
            messages: [{ role: "user", content: prompt }],
            max_tokens: 10,
            n: 1,
            stop: null,
            temperature: 0.5,
          }),
        }
      );
      const data = await response.json();
      if (data.choices && data.choices.length > 0) {
        return data.choices[0].message.content.trim();
      } else {
        console.error("Unexpected response format from OpenAI API:", data);
        return "Unknown";
      }
    } catch (error) {
      console.error("Error categorizing email:", error);
      return "Unknown";
    }
  };

  const summarizeContent = async () => {
    summaryLoading = true;
    summarizeDialog = true;

    const apiKey = env.PUBLIC_OPENAI_API_KEY;
    const filteredContent = content.filter(
      (item) =>
        item &&
        (selectedCategory === "all" ||
          (item.category !== "Spam/Advertisement" &&
            item.category === selectedCategory))
    );
    const contentText = filteredContent
      .map((item) => `Subject: ${item.subject}\nSnippet: ${item.snippet}`)
      .join("\n\n");

    const prompt = `Please summarize the following email content and be freindly to the user and engaging and include emojis:\n\n${contentText}\n\nSummary:`;

    try {
      const response = await fetch(
        "https://api.openai.com/v1/chat/completions",
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${apiKey}`,
          },
          body: JSON.stringify({
            model: "gpt-3.5-turbo",
            messages: [{ role: "user", content: prompt }],
            max_tokens: 150,
            n: 1,
            stop: null,
            temperature: 0.5,
          }),
        }
      );
      const data = await response.json();
      if (data.choices && data.choices.length > 0) {
        summary = data.choices[0].message.content.trim();
      } else {
        console.error("Unexpected response format from OpenAI API:", data);
        summary = "Failed to generate summary.";
      }
    } catch (error) {
      console.error("Error generating summary:", error);
      summary = "Failed to generate summary.";
    }

    summaryLoading = false;
  };
</script>

<main class="container mx-auto p-4">
  {#if gmailAuthenticated}
    <div>
      <RadioGroup.Root
        class="flex justify-between"
        bind:value={selectedCategory}
      >
        <div class="flex items-center space-x-2">
          <RadioGroup.Item value="all" id="all" />
          <Label for="all">All</Label>
        </div>
        <div class="flex items-center space-x-2">
          <RadioGroup.Item value="Fun/Social" id="fun-social" />
          <Label for="fun-social">Fun/Social</Label>
        </div>
        <div class="flex items-center space-x-2">
          <RadioGroup.Item value="Work" id="work" />
          <Label for="work">Work</Label>
        </div>
        <div class="flex items-center space-x-2">
          <RadioGroup.Item value="News/Info" id="news-info" />
          <Label for="news-info">News/Info</Label>
        </div>
      </RadioGroup.Root>
    </div>

    <div class="flex justify-center mt-4">
      <Dialog bind:open={summarizeDialog}>
        <DialogTrigger>
          <Button on:click={summarizeContent}>Summarize</Button>
        </DialogTrigger>
        <DialogContent>
          <DialogTitle>Summary</DialogTitle>
          {#if summaryLoading}
            <div class="flex justify-center">
              <p>Loading...</p>
            </div>
          {:else}
            <p>{summary}</p>
          {/if}
        </DialogContent>
      </Dialog>
    </div>

    <!-- Display fetched emails and content based on selected category -->
    {#each content.filter((item) => item && (selectedCategory === "all" || (item.category !== "Spam/Advertisement" && item.category === selectedCategory))) as item}
      <Card.Root class="my-4">
        <Card.Header class="flex justify-between">
          <Card.Title>{item.subject}</Card.Title>
          <Card.Title>{item.platform}</Card.Title>
        </Card.Header>
        <Card.Content>{item.snippet}</Card.Content>
        <Card.Footer>{item.category}</Card.Footer>
      </Card.Root>
    {/each}
    {#if content.length === 0}
      <div class="spinner"></div>
    {/if}
  {:else}
    <section
      class="hero relative p-8 text-center bg-cover bg-center min-h-[70vh] rounded-lg"
      style="background-image: url('/src/images/hero-image.webp');"
    >
      <div
        class="overlay absolute top-0 left-0 right-0 bottom-0 rounded-lg"
      ></div>
      <div class="relative z-10">
        <h1 class="text-8xl font-bold mb-4 text-white outlined-text">Aio</h1>
        <h2 class="text-4xl font-bold mb-4 text-white outlined-text">
          Transform Your Social Media Experience
        </h2>
        <p class="text-xl mb-8 text-white outlined-text">
          Manage your digital life with curated content tailored just for you.
        </p>
        <Button href="/settings" class="mx-0">Settings</Button>
      </div>
    </section>
  {/if}
</main>
