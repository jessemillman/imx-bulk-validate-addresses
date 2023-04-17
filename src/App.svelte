<script>
  import axios from "axios";
  import Papa from "papaparse";

  let file;
  let fileInput;
  let results = [];
  let progress = 0;
  let total = 0;
  let processing = false;
  const requestsPerSecond = 5;
  const delay = 1000 / requestsPerSecond;
  const batchSize = 5;

  async function fetchData(value, index) {
    const url = `https://api.x.immutable.com/v1/users/${value}`;
    try {
      const response = await axios.get(url);
      if (response.status === 200) {
        results[index].status = { text: "Registered on IMX", class: "success" };
      }
    } catch (error) {
      if (error.response && error.response.status === 404) {
        results[index].status = { text: "Unregistered on IMX", class: "false" };
        results[index].tooltip = JSON.stringify(error.response.data);
      }
    }
    progress++;
  }

  async function processBatch(batch) {
    const promises = batch.map(([value, index]) => fetchData(value, index));
    await Promise.all(promises);
  }

  const ETHEREUM_ADDRESS_REGEX = /^(0x)[0-9A-Fa-f]{40}$/;

  const processCSV = () => {
    if (file && file.type === "text/csv") {
      processing = true;
      Papa.parse(file, {
        header: false,
        complete: async (result) => {
          const data = result.data;
          total = data.length;
          for (const row of data) {
            const value = row[0];
            if (ETHEREUM_ADDRESS_REGEX.test(value)) {
              results.push({
                value,
                status: "pending",
                tooltip: "",
                valid: true,
              });
            } else {
              results.push({
                value,
                status: { text: "Invalid Address", class: "invalid" },
                tooltip: "",
                valid: false,
              });
              total--;
            }
          }

          for (let i = 0; i < results.length; i += batchSize) {
            const batch = results
              .slice(i, i + batchSize)
              .filter((row) => row.valid)
              .map((row, index) => [row.value, results.indexOf(row)]);
            if (batch.length > 0) {
              await processBatch(batch);
              await new Promise((resolve) => setTimeout(resolve, delay));
            }
          }
        },
      });
    }
  };

  let dragging = false;

  const handleDrop = (event) => {
    event.preventDefault();
    file = event.dataTransfer.files[0];
    processCSV();
    dragging = false;
  };

  const preventDefaultDrag = (event) => {
    event.preventDefault();
    event.stopPropagation();
  };
</script>

<main>
  <h1>Check IMX wallet registrations</h1>
  <!-- svelte-ignore a11y-click-events-have-key-events -->
  {#if !processing}
    <div
      class="drop-zone"
      class:dragging
      on:dragenter={() => (dragging = true)}
      on:dragleave={() => (dragging = false)}
      on:dragover={preventDefaultDrag}
      on:drop={handleDrop}
      on:click={() => fileInput.click()}
    >
      <p>Drag and drop a CSV file here</p>
    </div>
  {/if}

  {#if results.length}
    <table>
      <thead>
        <tr>
          <th>Address</th>
          <th>Status</th>
        </tr>
      </thead>
      <tbody>
        {#each results as result}
          <tr class={result.status.class}>
            <td>
              <a
                href="https://immutascan.io/address/{result.value}"
                target="_blank"
              >
                {result.value}
              </a>
            </td>
            <td>
              {#if result.status.class === "false"}
                <div class="tooltip">
                  {result.status.text}
                  <span class="tooltiptext">{result.tooltip}</span>
                </div>
              {:else}
                {result.status.text}
              {/if}
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  {/if}
  <div class="progress-bar" style="width: {(progress / total) * 100}%;" />
</main>

<style>
  main {
    font-family: "Suisseintl", sans-serif;
  }

  @font-face {
    font-family: "Suisseintl";
    src: url("https://assets.website-files.com/62535c6262b90afd768b9b26/629475d1b64554751267dd32_suisseintl-regular.woff2");
  }

  h1 {
    background-color: #fff;
    margin: 0 0 3vh 0;
    padding: 20px;
  }
  a {
    color: #f2f2f2;
  }
  table {
    border-collapse: collapse;
    width: 80%;
    margin: auto;
  }
  th,
  td {
    border: 1px solid black;
    padding: 20px;
    text-align: left;
  }
  th {
    background-color: #f2f2f2;
  }
  tr.success,
  tr.false {
    color: white;
  }
  tr.success {
    background-color: #82dd55;
  }
  tr.false {
    background-color: #e23636;
  }
  .progress-bar {
    position: fixed;
    bottom: 0;
    left: 0;
    height: 10px;
    background-color: #4caf50;
    transition: width 0.5s;
  }
  .tooltip {
    position: relative;
    display: inline-block;
    cursor: pointer;
  }

  .tooltip .tooltiptext {
    visibility: hidden;
    width: auto;
    min-width: 150px;
    background-color: #555;
    color: #fff;
    text-align: center;
    padding: 5px 10px;
    border-radius: 6px;
    position: absolute;
    z-index: 1;
    bottom: 125%;
    left: 50%;
    margin-left: -75px;
    opacity: 0;
    transition: opacity 0.3s;
  }

  .tooltip:hover .tooltiptext {
    visibility: visible;
    opacity: 1;
  }
  tr.invalid {
    background-color: #edb95e;
    color: black;
  }

  .drop-zone {
    border: 2px dashed #999;
    width: 80%;
    min-height: 30vh;
    margin: auto;
    border-radius: 6px;
    padding: 20px;
    text-align: center;
    cursor: pointer;
    background-color: #f8f8f8;
    margin-bottom: 20px;
    display: table;
  }
  .drop-zone p {
    display: table-cell;
    text-align: center;
    vertical-align: middle;
  }

  .drop-zone.dragging {
    border-color: #3f9eef;
    background-color: #eef8ff;
  }
</style>
