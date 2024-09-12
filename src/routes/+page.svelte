<script lang="ts">
  import Chart from "chart.js/auto";

  const colors = [
    "rgba(255, 99, 132, 0.2)",
    "rgba(54, 162, 235, 0.2)",
    "rgba(255, 206, 86, 0.2)",
    "rgba(75, 192, 192, 0.2)",
    "rgba(153, 102, 255, 0.2)",
    "rgba(255, 159, 64, 0.2)",
  ];

  let deposits = $state([
    {
      h2: 100,
      xh2: 100,
      hydro: 100,
      extra: 0,
      multiplier: 1,
      pendingRewards: 0,
      claimedRewards: 0,
    },
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
      claimedRewards: 0,
      pendingRewards: 0,
    });
  }

  function giveRewards(amount: number) {
    deposits = deposits.map((deposit) => {
      deposit.pendingRewards += (amount * deposit.hydro) / total.hydro;
      return deposit;
    });
  }
  function collectReward(index: number) {
    deposits[index].claimedRewards += deposits[index].pendingRewards;
    deposits[index].pendingRewards = 0;
  }

  function addH2(amount: number) {
    deposits = deposits.map((deposit) => {
      deposit.extra += (amount * deposit.hydro) / total.hydro;
      return deposit;
    });
  }

  let depositInput = $state(100);
  let addExtraH2Input = $state(1000);
  let giveRewardsInput = $state(1000);
  let multiplierInput = $state(1);

  let chartElement: HTMLCanvasElement;

  $effect(() => {
    const chart = new Chart(chartElement, {
      type: "bar",
      data: groupByAmount
        ? {
            labels: [
              "h2",
              "xh2",
              "hydro",
              "pendingRewards",
              "claimedRewards",
              "extra",
            ],
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
            datasets: [
              "h2",
              "xh2",
              "hydro",
              "pendingRewards",
              "claimedRewards",
              "extra",
            ].map((label, i) => {
              return {
                label,
                data: deposits.map(
                  (deposit) => deposit[label as keyof typeof deposit]
                ),
                backgroundColor: colors[i % colors.length],
                borderWidth: 1,
              };
            }),
          },
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
    return () => {
      chart.destroy();
    };
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

<!-- Form to give rewards -->
<form
  onsubmit={(e) => {
    e.preventDefault();
    giveRewards(giveRewardsInput);
  }}
>
  <label for="give-rewards-input">Give Rewards</label>
  <input type="number" id="give-rewards-input" bind:value={giveRewardsInput} />
  <button type="submit">Give Rewards</button>
</form>

Deposit details:
<div
  style="display: flex; flex-direction: row; gap: 1rem; align-items: flex-start;"
>
  <div style="">
    {#each deposits as deposit, i}
      <div class="position">
        <h3 class="title">Deposit {i}</h3>
        <div class="content">
          <div>H2<br />{deposit.h2}</div>
          <div>xH2<br />{deposit.xh2}</div>
          <div>HYDRO<br />{deposit.hydro}</div>
          <div>Extra<br />{deposit.extra}</div>
          <div>
            Pending Rewards<br />{deposit.pendingRewards}<br />
            <button onclick={() => collectReward(i)}>Collect Rewards</button>
          </div>
          <div>
            Collected Rewards<br />{deposit.claimedRewards}
          </div>
        </div>
      </div>
    {/each}
  </div>

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
  .position {
    flex: 0 0 400px;
  }
  .position .content {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 16px;
  }
</style>
