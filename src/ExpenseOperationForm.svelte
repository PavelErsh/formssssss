<script>
  import { onMount } from "svelte";

  const formatter = new Intl.NumberFormat("ru");
  const SECRET =
    "n5v0k1DkLVuLzGR2uftfh9bR1woL8PL6RlHeJBtLEk0lsA6KsiGYwRL3PvKOb4yCXZVPtjEsDeoipRV9s3sJ4q21AEpWwbo5ncixVKqz6iW0kLIjq035M5etxu0xEZnd";

  $: tg = null;

  let username = "";
  let maxDate = formatDate(new Date());
  let directories = {};

  let dateOperation = "";
  let selectedOperacion = 0;
  let selectedTypeTechnique = 0;
  $: countCanisters = null;
  $: stockCanisters =
    directories && directories.cost_fuels && selectedOperacion == 2
      ? directories.cost_fuels.reduce((a, b) => a + b.balance, 0)
      : selectedOperacion == 1
        ? countCanisters
        : 0;
  let amount = null;
  $: costFuel = calculateCostFuels(selectedOperacion, amount, countCanisters);
  let selectedTecnique = 0;
  let description = "";

  $: visibleTypeTechnique =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_type_technique
      : false;
  $: visibleCanisters =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_canisters
      : false;
  $: visibleAmount =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_amount
      : false;
  $: visibleCostFuel =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_cost_fuel
      : false;
  $: visibleTechnique =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_technique
      : false;
  $: visibleDescriptin =
    directories && directories.type_operations && directories.type_operations.find((r) => r.id == selectedOperacion)
      ? directories.type_operations.find((r) => r.id == selectedOperacion).visible_description
      : false;

  $: {
    if (dateOperation) {
      if (dateOperation > maxDate) dateOperation = maxDate;
    }
  }

  // Видимость кнопки сохранения
  $: tg && tg.MainButton
    ? (tg.MainButton.isVisible =
        /\d{4}-\d{2}-\d{2}/g.test(dateOperation) &&
        selectedOperacion > 0 &&
        (!visibleTypeTechnique ||
          (visibleTypeTechnique && selectedTypeTechnique > 0) ||
          !visibleTechnique ||
          (visibleTechnique && selectedTecnique > 0)) &&
        (!visibleCanisters ||
          (visibleCanisters && countCanisters && countCanisters > 0 && stockCanisters - countCanisters >= 0)) &&
        (!visibleAmount || (visibleAmount && amount && amount > 0)) &&
        (!visibleCostFuel || (visibleCostFuel && costFuel > 0)))
    : null;

  /**
   * @param {number} typeOperationID Идентификатор типа операции
   * @param {number} summ Стоимость приобретаемого топлива
   * @param {number} count Количество канистр
   */
  function calculateCostFuels(typeOperationID, summ, count) {
    switch (typeOperationID) {
      case 1: // Покупка бензина
        if (summ && count && summ > 0 && count > 0) return summ / count;
        return 0;
      case 2: // Заправка бензина
        let oilCoef = 0;
        if (directories && directories.oil_coefs && directories.oil_coefs.length) {
          let oc = directories.oil_coefs.find((r) => r.type_id == selectedTypeTechnique);
          if (oc) {
            oilCoef = oc.coef;
          }
        }

        if (
          directories &&
          directories.cost_fuels &&
          directories.cost_fuels.length &&
          count &&
          count > 0 &&
          stockCanisters - count >= 0
        ) {
          let allPrice = 0;
          for (let i = 0; i < directories.cost_fuels.length; i++) {
            let gl = directories.cost_fuels[i];
            if (gl.balance >= count) {
              allPrice += count * (gl.cost + oilCoef);
              return allPrice;
            } else {
              allPrice += gl.balance * (gl.cost + oilCoef);
              count -= gl.balance;
            }
          }
        }
        return 0;
    }
    return 0;
  }

  function formatDate(date) {
    let d = new Date(date),
      month = "" + (d.getMonth() + 1),
      day = "" + d.getDate(),
      year = d.getFullYear();

    if (month.length < 2) month = "0" + month;
    if (day.length < 2) day = "0" + day;

    return [year, month, day].join("-");
  }

  async function getServerDirectories() {
    fetch("/api/directories", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
      }),
    })
      .then((response) => {
        return response.json();
      })
      .then((data) => {
        directories = data;
      })
      .catch((err) => {
        console.error(err);
      });
  }

  async function sendServerData() {
    tg.MainButton.isVisible = false;

    let data = {
      dt: `${dateOperation}T00:00:00+03:00`,
      op_id: selectedOperacion,
      type_tech_id: selectedTypeTechnique,
      canisters: countCanisters ? countCanisters : 0,
      amount: amount ? amount : 0,
      cost_fuel: costFuel,
      tech_id: selectedTecnique,
      note: description,
      source_id: 1,
    };

    fetch("/api/operation", {
      method: "POST",
      headers: { "Content-Type": "application/json;charset=utf-8" },
      body: JSON.stringify({
        key: SECRET,
        value: tg.initData,
        data: data,
      }),
    })
      .catch((err) => {
        console.error(err);
      })
      .finally(() => tg.close());
  }

  // Загружаем данные при создании формы
  onMount(() => {
    getServerDirectories();
    // getMinDate();
    dateOperation = formatDate(new Date());

    tg = window.Telegram.WebApp;

    if (tg && tg.initDataUnsafe && tg.initDataUnsafe.user) {
      username = tg.initDataUnsafe.user.username;
    }

    if (tg && tg.MainButton) {
      tg.MainButton.setText("Сохранить");
      tg.MainButton.color = "#a3e635";
      tg.backgroundColor = "#e5e7eb";
      tg.headerColor = "#e5e7eb";
    }

    tg.onEvent("mainButtonClicked", () => sendServerData());
    tg.enableClosingConfirmation();
  });
</script>

<main class="max-w-lg bg-gray-200">
  <div class="pb-64 w-full px-3">
    <div class="m-1">
      <h2 class="ml-3 text-3xl font-bold">Операция</h2>
    </div>

    <div class="m-1 text-2xl">
      <div class="mt-2 grid grid-cols-1 gap-1.5">
        <!-- Дата операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Дата</span>
          <input
            bind:value={dateOperation}
            type="date"
            class="mt-1 block h-10 w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          />
        </label>

        <!-- Вид операции -->
        <label class="block">
          <span class="ml-1 text-gray-700">Вид операции</span>
          <select
            bind:value={selectedOperacion}
            class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
          >
            <option value="" disabled selected></option>
            {#if directories && directories.type_operations}
              {#each directories.type_operations as oper}
                <option value={oper.id}>{oper.name}</option>
              {/each}
            {/if}
          </select>
        </label>

        {#if selectedOperacion == 1 && directories.canisters > 0}
          <span class="ml-1 text-red-900 text-xl">Имеются не использованные канистры: {directories.canisters}</span>
        {:else if selectedOperacion == 2 && directories.canisters == 0}
          <span class="ml-1 text-red-900 text-xl">Отсутствуют канистры с топливом</span>
        {:else}
          <!-- Группа техники -->
          {#if visibleTypeTechnique}
            <label class="block">
              <span class="ml-1 text-gray-700">Группа техники</span>
              <select
                bind:value={selectedTypeTechnique}
                disabled={(selectedOperacion == 4 || selectedOperacion == 5) && selectedTecnique > 0}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              >
                <option value="0" selected></option>
                {#if directories && directories.type_techniques}
                  {#each directories.type_techniques as typeTech}
                    <option value={typeTech.id}>{typeTech.name}</option>
                  {/each}
                {/if}
              </select>
            </label>
          {/if}

          <!-- Вид техники -->
          {#if visibleTechnique}
            <label class="block">
              <span class="ml-1 text-gray-700">Вид техники</span>
              <select
                bind:value={selectedTecnique}
                disabled={(selectedOperacion == 4 || selectedOperacion == 5) && selectedTypeTechnique > 0}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              >
                <option value="0" selected></option>
                {#if directories && directories.techniques}
                  {#each directories.type_techniques as typeTech, index}
                    <optgroup label={typeTech.name} class={index == 1 ? "bg-orange-100" : index == 2 ? "bg-violet-100" : ""}>
                      {#each directories.techniques.filter((t) => t.type_id == typeTech.id) as tech}
                        <option value={tech.id}>{tech.name}</option>
                      {/each}
                    </optgroup>
                  {/each}
                {/if}
              </select>
            </label>
          {/if}

          <!-- Количество канистр -->
          {#if visibleCanisters}
            <label class="block">
              <span class="ml-1 text-gray-700">Количество канистр</span>
              <input
                bind:value={countCanisters}
                type="number"
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              />
              {#if selectedOperacion == 2}
                {#if stockCanisters - countCanisters > 0}
                  <span class="ml-1 text-green-500 text-lg">В наличии еще {stockCanisters - countCanisters} канистр</span>
                {:else if stockCanisters - countCanisters < 0}
                  <span class="ml-1 text-red-500 text-lg">Не хватает канистр: {stockCanisters - countCanisters}</span>
                {:else}
                  <span class="ml-1 text-gray-500 text-lg">Канистры закончились</span>
                {/if}
              {/if}
            </label>
          {/if}

          <!-- Сумма -->
          {#if visibleAmount}
            <label class="block">
              <span class="ml-1 text-gray-700">Сумма</span>
              <input
                bind:value={amount}
                type="number"
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
              />
            </label>
          {/if}

          <!-- Стоимость бензина -->
          {#if visibleCostFuel && costFuel > 0}
            <span class="ml-1 text-gray-500"
              >{selectedOperacion == 1
                ? "Стоимость канистры: "
                : selectedOperacion == 2
                  ? "Стоимость топлива: "
                  : ""}{formatter.format(costFuel)} ₽</span
            >
          {/if}

          <!-- Примечание -->
          {#if visibleDescriptin}
            <label class="block">
              <span class="ml-1 text-gray-700">Комментарий</span>
              <textarea
                bind:value={description}
                class="mt-1 block w-full rounded border-gray-300 py-1 text-xl shadow-sm focus:border-gray-500 focus:ring focus:ring-gray-400 focus:ring-opacity-50"
                rows="3"
              />
            </label>
          {/if}
        {/if}
      </div>
    </div>
  </div>
