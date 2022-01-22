<template>
  <div class="p-8">
    <h1 class="text-gray-50 text-3xl font-bold mb-4">
      Number Provisioning Utility
    </h1>
    <div class="mb-8">
      <label class="block mb-2 text-blue-300 font-semibold" for="input">
        Raw List
      </label>
      <textarea
        id="input"
        name="input"
        class="bg-gray-800 border border-gray-700 rounded p-4 w-full max-w-xl h-64 focus:outline-none focus:border-blue-500 hover:border-blue-500"
        @input="parseRawList($event.target.value)"
      />
    </div>
    <div class="mb-8">
      <div v-if="provisioningListErrors.length">
        <div
          class="text-red-200 bg-red-900 p-3 rounded border border-red-600 mb-5"
        >
          <h3 class="font-bold mb-2">Errors</h3>
          <ul>
            <li v-for="error in provisioningListErrors">
              {{ error }}
            </li>
          </ul>
        </div>
      </div>
      <div class="space-y-8">
        <div
          v-for="(accounts, globalAreaCode) in accountsByArea"
          :key="globalAreaCode"
        >
          <h3 class="text-xl mb-2 text-blue-50">
            <span class="font-bold">{{ globalAreaCode }}</span> &ndash;
            {{ globalAreaCodeNames[globalAreaCode] }} &ndash;
            {{ accounts.length }}
          </h3>
          <div class="space-y-2">
            <div>
              <label
                class="block mb-2 text-blue-300 font-semibold"
                :for="`input-${globalAreaCode}`"
              >
                Numbers
              </label>
              <textarea
                :id="`input-${globalAreaCode}`"
                :name="`input-${globalAreaCode}`"
                class="bg-gray-800 border border-gray-700 rounded p-4 w-full max-w-xl h-32 focus:outline-none focus:border-blue-500 hover:border-blue-500"
                @input="
                  updateLines($event.target.value, accounts, globalAreaCode)
                "
              />
            </div>
            <div>
              <label
                class="block text-blue-300 font-semibold"
                :for="`input-${globalAreaCode}`"
              >
                Lines
              </label>
              <textarea
                :id="`output-${globalAreaCode}`"
                :name="`output-${globalAreaCode}`"
                class="bg-gray-800 border border-gray-700 rounded p-4 w-full max-w-xl h-32 focus:outline-none focus:border-blue-500 hover:border-blue-500"
                :value="formatLines(accounts)"
                readonly
              />
            </div>
          </div>
        </div>
      </div>
    </div>
    <div>
      <h2 class="text-xl mb-4 font-bold text-blue-50">
        Number Provising Output
      </h2>
      <label class="block mb-2 text-blue-300 font-semibold" for="input">
        Final List
      </label>
      <textarea
        id="input"
        name="input"
        class="bg-gray-800 border border-gray-700 rounded p-4 w-full max-w-xl h-64 focus:outline-none focus:border-blue-500 hover:border-blue-500"
        ref="finalListInput"
      />
    </div>
  </div>
</template>

<script setup>
import { ref } from "vue";
import postcodeMapping from "./assets/postcodeMapping.json";

const date = new Date();
const today = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;

/** @type {import("vue").Ref<{ [area: string]: { provisionedNumber?: string, accountNumber: number, salesOrder: number, postCode: string, numbersRequired: number }[] }>} */
const accountsByArea = ref({});

const provisioningListErrors = ref([]);
const globalAreaCodeNames = ref({});
const finalListInput = ref();

/**
 * @param {string} input
 */
function parseRawList(text) {
  const regex =
    /^(\d+)\s+(\d+)\s+(?:Line\s+\d\s+Missing\s+){1,2}([a-z]{1,2}\d{1,2}\s\d[a-z]{2})$/i;

  const lines = text
    .split("\n")
    .map((value) => value.trim())
    .filter((value) => value);

  /** @type {{ [area: string]: { accountNumber: number, salesOrder: number, postCode: string, numbersRequired: number }[] }} */
  const accountsPerAreaCode = {};

  provisioningListErrors.value = [];

  for (const [index, line] of lines.entries()) {
    try {
      const match = line.match(regex);

      if (!match) {
        throw new Error(`Unable to parse "${line}" on line ${index + 1}`);
      }

      const [, accountNumber, salesOrder, postCode] = match;

      const postCodeArea = postCode.match(/^([a-z0-9]+)/i);

      if (!postCodeArea) {
        throw new Error(
          `${accountNumber} (on line ${index + 1}) has invalid postcode`
        );
      }

      if (!postcodeMapping[postCodeArea[0]]) {
        throw new Error(
          `${accountNumber} (on line ${index + 1}) has unknown postcode area: ${
            postCodeArea[0]
          }`
        );
      }

      const [areaName, areaCode] = postcodeMapping[postCodeArea[0]];

      // TODO: check if numbers are correct, not e.g. line 1 and line 1 missing
      const numbersRequired = [...line.matchAll(/Line\s+(\d)\s+Missing/gi)];

      if (numbersRequired.length === 0 || numbersRequired.length > 2) {
        throw new Error(
          `${accountNumber} (on line ${
            index + 1
          }) wrong number of "line missing" values`
        );
      }

      const globalAreaCode = areaCode.replace(/^0/, "44");

      globalAreaCodeNames.value[globalAreaCode] = areaName;

      if (!accountsPerAreaCode[globalAreaCode]) {
        accountsPerAreaCode[globalAreaCode] = [];
      }

      for (const [, lineNumber] of numbersRequired) {
        accountsPerAreaCode[globalAreaCode].push({
          accountNumber,
          salesOrder,
          postCode,
          lineNumber,
          globalAreaCode,
          areaCode,
          areaName,
        });
      }
    } catch (error) {
      provisioningListErrors.value.push(error.message);
    }
  }

  // value = element
  finalListInput.value.value = "";

  accountsByArea.value = accountsPerAreaCode;
}

/**
 * @param {string} input
 * @param {object[]} accounts
 * @param {string} globalAreaCode
 */
function updateLines(text, accounts, globalAreaCode) {
  const lines = text
    .split("\n")
    .map((value) => value.trim())
    .filter((value) => value);

  accountsByArea.value = {
    ...accountsByArea.value,
    [globalAreaCode]: accounts.map((account, index) => ({
      ...account,
      provisionedNumber: lines[index] || "",
    })),
  };

  const input = document.getElementById(`output-${globalAreaCode}`);

  input.value = accountsByArea.value[globalAreaCode]
    .map(
      ({ provisionedNumber, accountNumber, salesOrder }) =>
        `${provisionedNumber},${accountNumber},${salesOrder},${today}`
    )
    .join("\n");

  updateFinalList();
}

/**
 * @param {{ provisionedNumber?: string, accountNumber: string, salesOrder: string }[]} accounts
 */
function formatLines(accounts) {
  return accounts
    .map(
      ({ provisionedNumber, accountNumber, salesOrder }) =>
        `${provisionedNumber || ""},${accountNumber},${salesOrder},${today}`
    )
    .join("\n");
}

function updateFinalList() {
  finalListInput.value.value = Object.values(accountsByArea.value)
    .flat()
    .map(
      ({ accountNumber, salesOrder, provisionedNumber, lineNumber }) =>
        `${accountNumber} ${salesOrder} ${provisionedNumber || ""} ${lineNumber} NEW`
    )
    .join("\n");
}
</script>
