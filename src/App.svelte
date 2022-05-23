<script>
    import * as zip from "@zip.js/zip.js";
    console.log(zip);

    let bookChapters = null;

    let currentChapter = 0;
    let currentWord = "--";
    let previousWord = "--";
    let nextWord = "--";

    let chapterTitle = "";
    let chapterProgress = 0;

    let chapterTime = 0;
    let chapterTimeTaken = 0;

    let msPerChar = 20;
    let minimumMsPerWord = 100;


    const timeout = async (ms) => new Promise((res) => setTimeout(res, ms));

    const handleFile = async (e) => {
        let chapters = [];

        const file = e.target.files[0];
        const entries = await new zip.ZipReader(
            new zip.BlobReader(file)
        ).getEntries();

        console.log(entries);

        const docs = entries.filter((e) =>
            /*e.filename.startsWith("OEBPS/Text/") &&*/ e.filename.endsWith("html") && !e.directory
        ).sort((a, b) => a.filename.localeCompare(b.filename));

        console.log(docs);

        for (const doc of docs) {
            const content = await doc.getData(new zip.TextWriter());
            const chapter = content.toString().replace("&nbsp;", " ");

            const parser = new DOMParser();
            const xmlDoc = parser.parseFromString(chapter, "text/xml");

            let title;
            if (xmlDoc.getElementsByTagName("h1")[0]) {
                title = xmlDoc.getElementsByTagName("h1")[0].textContent;
            } else if (xmlDoc.getElementsByTagName("title")[0]) {
                title = xmlDoc.getElementsByTagName("title")[0].textContent;
            } else {
                title = "Untitled";
            }

            const body = xmlDoc.getElementsByTagName("body")[0].innerText;

            chapters.push({
                title,
                body,
            });
        }

        window["chapters"] = chapters;
        bookChapters = chapters;

        for (const [index, chapter] of bookChapters.entries()) {

          
            currentChapter = index;
            currentWord = "";

            chapterTitle = chapter.title;
            if (!chapterTitle.toLowerCase().includes("chapter")) {
                chapterTitle = `Chapter ${index + 1} - ${chapterTitle}`;
            }

            const words = chapter.body.split(" ");
            chapterTime = words.map((word) => (word.length > 5 ? word.length * msPerChar : minimumMsPerWord)).reduce((a, b) => a + b, 0);
            chapterTimeTaken = 0;

            for (const [word_index, word] of words.entries()) {
                previousWord = currentWord;
                currentWord = word;
                nextWord = words[word_index + 1] || "--";

                let delay = msPerChar * word.length;
                if (delay < minimumMsPerWord) {
                    delay = minimumMsPerWord;
                }
                
                chapterProgress = word_index / words.length;
                chapterTimeTaken += delay;

                await timeout(delay);
            }
        }
    };
</script>

<main>

  <div class="logotype">
    <h1>spdrd.</h1>
  </div>

  {#if bookChapters}

      <div class="words">
          <div class="sub prev">{previousWord}</div>
          <div class="main current">{currentWord}</div>
          <div class="sub next">{nextWord}</div>
      </div>

      <div class="bottom-chapter">
        <div class="chapter-title">{chapterTitle}</div>
        <div class="chapter-index">{currentChapter + 1} / {bookChapters.length} - {Math.round((chapterTime - chapterTimeTaken)/1000)}s remaining</div>

        <div style={`width: ${chapterProgress * 100}%`} class="progress-bar"></div>
      </div>
  {:else}

      <h1>spdrd helps you read faster.</h1>
      <h2>choose an .epub file</h2>

      <input type="file" on:change={handleFile} accept=".epub" />

      <label>
        Milliseconds per character ({msPerChar}) <input type="range" min="0" max="100" bind:value={msPerChar} step="1" />
      </label>

  {/if}
</main>

<style>

  main {
    position: relative;

    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 100vh;
    text-align: center;

    font-family: 'Helvetica Neue', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }

  h1, h2 {
    margin: 10px;
  }

  input {
    margin-top: 2rem;
  }

  .logotype {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    z-index: 1;

    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }

  .logotype h1 {
    margin: 10px;
    font-size: 1em;
  }

  .words {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-flow: column nowrap;
    gap: 1em;
    font-family: Georgia, 'Times New Roman', Times, serif;
  }

  .words .sub {
    font-size: 0.8em;
    font-style: italic;
  }

  .words .main {
    font-size: 5em;
    font-weight: bold;
  }

  .bottom-chapter {
    display: flex;
    flex-flow: column nowrap;
    align-items: center;
    justify-content: center;

    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
    padding: 1rem;
  }

  .bottom-chapter .chapter-title {
    font-size: 1em;
    font-weight: bold;
  }

  .bottom-chapter .chapter-index {
    font-size: 0.8em;
    /* font-style: italic; */
  }

  .bottom-chapter .progress-bar {
    width: 0%;
    height: 100%;

    z-index: -1;

    background-color: #eee;

    position: absolute;
    bottom: 0;
    left: 0;

    transition: width 0.5s;
  }

</style>