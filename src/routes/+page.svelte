<script lang="ts">
  import { Chart, registerables } from "chart.js";

  const colors = [
    "rgba(255, 99, 132, 0.2)",
    "rgba(54, 162, 235, 0.2)",
    "rgba(255, 206, 86, 0.2)",
    "rgba(75, 192, 192, 0.2)",
  ];

  let deposits = $state([
    { h2: 100, xh2: 100, hydro: 100, extra: 0, multiplier: 1 },
  ]);

  let total = $derived(
    deposits.reduce(
      (totals, deposit) => {
        totals.h2 += deposit.h2 + deposit.extra;
        totals.xh2 += deposit.xh2;
        totals.hydro += deposit.hydro;
        return totals;
      },
      { h2: 0, xh2: 0, hydro: 0 }
    )
  );
  let rate = $derived(total.h2 / total.xh2);

  let groupByAmount = $state(false);
  let stacked = $state(false);

  function deposit(h2: number, multiplier: number) {
    const xh2 = h2 / rate;
    const hydro = h2 * multiplier;
    deposits.push({
      h2,
      xh2,
      hydro,
      multiplier,
      extra: 0,
    });
  }
  function addH2(amount: number) {
    deposits = deposits.map((deposit) => {
      deposit.extra += (amount * deposit.hydro) / total.hydro;
      return deposit;
    });
  }

  let depositInput = $state(100);
  let addExtraH2Input = $state(1000);
  let multiplierInput = $state(1);

  let chartElement: HTMLCanvasElement;

  let chart: Chart;

  Chart.register(...registerables);

  $effect(() => {
    const data = groupByAmount
      ? {
          labels: ["H2", "xH2", "HYDRO", "EXTRA"],
          datasets: deposits.map((deposit, i) => {
            return {
              label: "Deposits " + i,
              data: [deposit.h2, deposit.xh2, deposit.hydro, deposit.extra],
              backgroundColor: colors[i % colors.length],
              borderWidth: 1,
            };
          }),
        }
      : {
          // group by deposit
          labels: deposits.map((deposit, i) => `Deposit ${i}`),
          datasets: ["h2", "xh2", "hydro", "extra"].map((label, i) => {
            return {
              label,
              data: deposits.map(
                (deposit) => deposit[label as keyof typeof deposit]
              ),
              backgroundColor: colors[i % colors.length],
              borderWidth: 1,
            };
          }),
        };

    if (!chart) {
      const ctx = document.getElementById("chart") as any;
      chart = new Chart(ctx, {
        type: "bar",
        data,
        options: {
          animation: false,
          responsive: true,
          scales: {
            y: {
              stacked,
              beginAtZero: true,
            },
            x: {
              stacked,
            },
          },
        },
      });
    } else {
      chart.data = data;
      (chart.options.scales as any).y.stacked = stacked;
      (chart.options.scales as any).x.stacked = stacked;
      chart.update();
    }
  });
</script>

<h1>H2 Vault Demo</h1>

<p>Deposited H2 Amount: {total.h2}</p>
<p>Deposited xh2 Amount: {total.xh2}</p>
<p>Current Rate H2 : xh2 = {rate.toFixed(4)}</p>

<!-- Form to deposit H2 -->
<form
  onsubmit={(e) => {
    e.preventDefault();
    deposit(depositInput, multiplierInput);
  }}
>
  <label for="deposit-input">Deposit H2</label>
  <input type="number" id="deposit-input" bind:value={depositInput} />
  <label for="multiplier-input">Multiplier</label>
  <input type="number" id="multiplier-input" bind:value={multiplierInput} />
  <button type="submit">Deposit</button>
  You will get {depositInput} / {rate.toFixed(2)} = {(
    depositInput / rate
  ).toFixed(2)} xH2 and {(depositInput * multiplierInput).toFixed(2)} HYDRO
</form>
<!-- Form to add extra h2 -->
<form
  onsubmit={(e) => {
    e.preventDefault();
    addH2(addExtraH2Input);
  }}
>
  <label for="add-extra-h2-input">Add Extra H2</label>
  <input type="number" id="add-extra-h2-input" bind:value={addExtraH2Input} />
  <button type="submit">Add Extra H2</button>
</form>

Deposit details:
<div
  style="display: flex; flex-direction: row; gap: 1rem; align-items: flex-start;"
>
  <table class="deposits" style="flex:1">
    <thead>
      <tr>
        <th>Index</th>
        <th>H2</th>
        <th>xh2</th>
        <th>hydro</th>
        <th>multiplier</th>
        <th>extra</th>
      </tr>
    </thead>
    <tbody>
      {#each deposits as deposit, i}
        <tr>
          <td>{i}</td>
          <td>{deposit.h2.toFixed(2)}</td>
          <td>{deposit.xh2.toFixed(2)}</td>
          <td>{deposit.hydro.toFixed(2)}</td>
          <td>{deposit.multiplier}</td>
          <td>{deposit.extra.toFixed(2)}</td>
        </tr>
      {/each}
    </tbody>
  </table>

  <div style="flex: 1">
    <div>
      <label for="group-by-amount">Group by amount</label>
      <input
        type="checkbox"
        id="group-by-amount"
        bind:checked={groupByAmount}
      />
      <label for="stacked">Stacked</label>
      <input type="checkbox" id="stacked" bind:checked={stacked} />
    </div>
    <canvas bind:this={chartElement} id="chart" height="200"></canvas>
  </div>
</div>

<style>
  .deposits {
    border-collapse: collapse;
  }
  .deposits th,
  .deposits td {
    width: 100px;

    padding: 0.5rem;
    text-align: left;
  }
</style>
