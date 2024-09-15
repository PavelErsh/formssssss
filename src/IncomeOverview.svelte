<script>
  import { onMount } from "svelte";
  import Chart from "chart.js/auto";

  const formatter = new Intl.NumberFormat("ru");
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  const months = [
    "январь",
    "февраль",
    "март",
    "апрель",
    "май",
    "июнь",
    "июль",
    "август",
    "сентябрь",
    "октябрь",
    "ноябрь",
    "декабрь",
  ];
  const nameMonths = [
    "января",
    "февраля",
    "марта",
    "апреля",
    "мая",
    "июня",
    "июля",
    "августа",
    "сентября",
    "октября",
    "ноября",
    "декабря",
  ];

  let years = [2024];

  let username = "";
  let selectedYear = 2024;
  let selectedMonth;
  let fullUsername = "";
  let salaryDays = [];
  let filteredSalary = [];

  let summury = { salary: 0, percent: 0, penalty: 0 };

  $: calc(selectedYear, selectedMonth, periodChart);

  let periodChart = "month";

  let data = {
    labels: [],
    datasets: [
      {
        label: "Штрафы",
        data: [],
        borderColor: "hsl(0, 100%, 80%)",
        backgroundColor: "hsla(0, 100%, 80%, 0.6)",
        tension: 0.4,
        pointRadius: 5,
        yAxisID: "y",
      },
      {
        label: "Процент",
        data: [],
        borderColor: "hsl(120, 100%, 70%)",
        backgroundColor: "hsla(120, 100%, 70%, 0.6)",
        tension: 0.4,
        pointRadius: 0,
        yAxisID: "y",
      },
      {
        label: "Оклад",
        data: [],
        borderColor: "hsl(210, 100%, 60%)",
        backgroundColor: "hsla(210, 100%, 60%, 0.6)",
        tension: 0.4,
        pointRadius: 0,
        yAxisID: "y",
      },
    ],
  };

  const chartConfig = {
    type: "line",
    data: data,
    options: {
      responsive: true,
      interaction: {
        mode: "index",
        intersect: false,
      },
      plugins: {
        legend: {
          display: false,
        },
      },
      scales: {
        x: {
          display: false,
          grid: {
            drawOnChartArea: false,
          },
        },
        y: {
          type: "linear",
          display: false,
          position: "left",
          grid: {
            drawOnChartArea: false,
          },
        },
      },
    },
  };

  let chartCanvas;
  let chart;

  const currentDate = new Date();

  function calc(year, month, period) {
    if (chart) chart.destroy();

    if (period == "month") {
      data.labels = [];
      data.datasets[0].data = [];
      data.datasets[1].data = [];
      data.datasets[2].data = [];
    } else {
      data.labels = months.map((m) => m.slice(0, 1).toUpperCase() + m.slice(1));
      filteredSalary = new Array(12);
      data.datasets[0].data = new Array(12);
      data.datasets[1].data = new Array(12);
      data.datasets[2].data = new Array(12);
    }

    summury = { salary: 0, percent: 0, penalty: 0 };
    filteredSalary = [];
    salaryDays.forEach((d) => {
      if (period == "month" && d.year == year && d.month == month) {
        summury.salary += d.salary;
        summury.percent += d.percent;
        summury.penalty += d.penalty;

        data.labels.push(`${("0" + d.day).slice(-2)}.${("0" + (d.month + 1)).slice(-2)}.${d.year}`);
        data.datasets[0].data.push(d.penalty > 0 ? d.penalty : null);
        data.datasets[1].data.push(d.percent > 0 ? d.percent : null);
        data.datasets[2].data.push(d.salary > 0 ? d.salary : null);

        filteredSalary.push({
          period: `${d.day} ${nameMonths[d.month]}`,
          penalty: d.penalty,
          percent: d.percent,
          salary: d.salary,
        });
      } else if (period == "year" && d.year == year) {
        summury.salary += d.salary;
        summury.percent += d.percent;
        summury.penalty += d.penalty;

        if (!data.datasets[0].data[d.month]) data.datasets[0].data[d.month] = d.penalty;
        else data.datasets[0].data[d.month] += d.penalty;
        if (!data.datasets[1].data[d.month]) data.datasets[1].data[d.month] = d.percent;
        else data.datasets[1].data[d.month] += d.percent;
        if (!data.datasets[2].data[d.month]) data.datasets[2].data[d.month] = d.salary;
        else data.datasets[2].data[d.month] += d.salary;

        if (!filteredSalary[d.month]) {
          filteredSalary[d.month] = {
            period: months[d.month].slice(0, 1).toUpperCase() + months[d.month].slice(1),
            penalty: d.penalty,
            percent: d.percent,
            salary: d.salary,
          };
        } else {
          filteredSalary[d.month].penalty += d.penalty;
          filteredSalary[d.month].percent += d.percent;
          filteredSalary[d.month].salary += d.salary;
        }
      }
    });

    filteredSalary = filteredSalary;

    if (chartCanvas) {
      let ctx = chartCanvas.getContext("2d");
      chart = new Chart(ctx, chartConfig);
    }
  }

  async function getServerSalary() {
    fetch("/api/user_salary", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
        username,
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        salaryDays = data.salary_days;
        fullUsername = data.user.name;
        years = data.years;

        const currentDate = new Date();
        selectedYear = currentDate.getFullYear();
        selectedMonth = currentDate.getMonth();
      })
      .catch((err) => {
        console.error(err);
      });
  }

  // Загружаем данные при создании формы
  onMount(() => {
    tg = window.Telegram.WebApp;
    if (tg && tg.initDataUnsafe && tg.initDataUnsafe.user) {
      username = tg.initDataUnsafe.user.username;
    }

    getServerSalary();

    if (tg && tg.MainButton) {
      tg.MainButton.setText("Закрыть");
      tg.MainButton.color = "#a3e635";
      tg.backgroundColor = "#e5e7eb";
      tg.headerColor = "#e5e7eb";
    }
  });
</script>

<main class="max-w-lg bg-gray-200">
  <div class="w-full px-2 pb-64">
    <div class="grid grid-cols-1 gap-1.5">
      <div class="grid grid-cols-4 grid-rows-2">
        <span class="row-span-2 text-gray-700">
          <img class=" w-16 m-2 pt-3" src="extrim.png" alt="ЭКСТРИМ АРХЫЗ" />
        </span>
        <span class="text-gray-700">Сотрудник</span>
        <span class="text-gray-700">Год</span>
        <span class="text-gray-700">Месяц</span>
        <span class="text-sm">{fullUsername}</span>
        <select
          bind:value={selectedYear}
          class="w-full text-sm rounded border-gray-300 px-2 py-0.5 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
        >
          {#if years.length}
            {#each years as year}
              <option value={year} class={year == currentDate.getFullYear() ? "selected" : ""}>{year}</option>
            {/each}
          {/if}
        </select>
        <select
          bind:value={selectedMonth}
          class="w-full text-sm rounded border-gray-300 px-2 py-0.5 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50"
        >
          {#if months.length}
            {#each months as month, index}
              <option value={index} class={index == currentDate.getMonth() ? "selected" : ""}>{month}</option>
            {/each}
          {/if}
        </select>
      </div>
    </div>

    <div class="mt-1 flex justify-stretch gap-1 border-b-4 border-t-4 border-sky-400">
      <div class="w-40">
        <table class="table-auto border-collapse">
          <tbody>
            <tr>
              <td
                ><input
                  type="checkbox"
                  class="border-blue-gray-200 -mt-1 h-5 w-5 rounded ring-0 focus:ring-0"
                  checked
                  on:change={() => {
                    chart.setDatasetVisibility(2, !chart.isDatasetVisible(2));
                    chart.update();
                  }}
                /></td
              >
              <td class="border border-b-sky-400 px-3">Оклад</td>
              <td class="border border-b-sky-400 text-right">{formatter.format(summury.salary)}</td>
            </tr>
            <tr>
              <td
                ><input
                  type="checkbox"
                  class="border-blue-gray-200 -mt-1 h-5 w-5 rounded ring-0 focus:ring-0"
                  checked
                  on:change={() => {
                    chart.setDatasetVisibility(1, !chart.isDatasetVisible(1));
                    chart.update();
                  }}
                /></td
              >
              <td class="border border-b-sky-400 px-3">Процент</td>
              <td class="border border-b-sky-400 text-right">{formatter.format(summury.percent)}</td>
            </tr>
            <tr>
              <td
                ><input
                  type="checkbox"
                  class="border-blue-gray-200 -mt-1 h-5 w-5 rounded ring-0 focus:ring-0"
                  checked
                  on:change={() => {
                    chart.setDatasetVisibility(0, !chart.isDatasetVisible(0));
                    chart.update();
                  }}
                /></td
              >
              <td class="px-3">Штраф</td>
              <td class="text-right">{formatter.format(summury.penalty)}</td>
            </tr>
          </tbody>
        </table>
        <div class="flex items-center px-2 py-1">
          <input
            class="border-blue-gray-200 h-4 w-4 rounded-full ring-0 focus:ring-0"
            type="radio"
            bind:group={periodChart}
            value="month"
          />
          <span class="ml-2 text-sm">Месяц</span>
          <input
            class="border-blue-gray-200 h-4 w-4 rounded-full ml-3 ring-0 focus:ring-0"
            type="radio"
            bind:group={periodChart}
            value="year"
          />
          <span class="ml-2 text-sm">Год</span>
        </div>
      </div>
      <div class="grow">
        <canvas class="h-20 grow object-fill" bind:this={chartCanvas} />
      </div>
    </div>

    <div class="mt-1 grid grid-cols-4 text-sm">
      <div class="col-span-4">Детализация</div>

      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Дата</div>
      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Оклад</div>
      <div class="border-b border-r border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Процент</div>
      <div class="border-b border-gray-200 bg-slate-300 py-0.5 text-center font-bold">Штраф</div>

      <div class="border-r border-r-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">Итого:</div>
      <div class="border-r border-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">
        {formatter.format(summury.salary)}
      </div>
      <div class="border-r border-gray-200 bg-gray-300 px-1 py-0.5 text-right font-bold">
        {formatter.format(summury.percent)}
      </div>
      <div class="bg-gray-300 px-1 py-0.5 text-right font-bold {summury.penalty > 0 ? 'bg-rose-300' : 'text-gray-500'}">
        {formatter.format(summury.penalty)}
      </div>

      {#if filteredSalary.length}
        {#each filteredSalary as p}
          <div class="border-b border-r border-gray-300 px-1">{p.period}</div>
          <div class="border-b border-r border-gray-300 px-1 text-right">{formatter.format(p.salary)}</div>
          <div class="border-b border-r border-gray-300 px-1 text-right">{formatter.format(p.percent)}</div>
          <div class="border-b border-gray-300 px-1 text-right {p.penalty > 0 ? 'bg-rose-300' : 'text-gray-400'}">
            {formatter.format(p.penalty)}
          </div>
        {/each}
      {/if}
    </div>
  </div>
</main>
