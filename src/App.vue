<template>
  <div class="p-8">
    <h1 class="text-gray-50 text-xl font-semibold mb-8">
      <span class="font-bold">Karl's</span> Number Provisioning Utility
    </h1>
    <div class="mb-8">
      <label class="block mb-2 text-blue-300 font-semibold" for="input">
        Raw List Input
      </label>
      <textarea
        id="input"
        name="input"
        class="bg-gray-800 border border-gray-700 rounded p-4 w-full max-w-xl h-64 focus:outline-none focus:border-blue-500 hover:border-blue-500"
        @input="parseRawList($event.target.value)"
      />
    </div>
    <div class="mb-8">
      <div class="block mb-2 text-blue-300 font-semibold">
        Allocation List Output
      </div>
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
      <pre>{{ provisioningListOutput }}</pre>
    </div>
    <div class="mb-8">
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

const provisioningListOutput = ref("");
const provisioningListErrors = ref([]);

const finalListInput = ref();
// const finalListOutput = ref("");
// const finalListErrors = ref([]);

/** @param {string} input */
function parseRawList(input) {
  const regex =
    /^(\d+)\s+(\d+)\s+(?:Line\s+\d\s+Missing\s+){1,2}([a-z]{1,2}\d{1,2}\s\d[a-z]{2})$/i;

  const lines = input
    .split("\n")
    .map((value) => value.trim())
    .filter((value) => value);

  /** @type {{ [area: string]: [{ accountNumber: number, salesOrder: number, postCode: string, numbersRequired: number }] }} */
  const accountsPerArea = {};
  const areaCodeNames = {};

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

      areaCodeNames[globalAreaCode] = areaName;

      if (!accountsPerArea[globalAreaCode]) {
        accountsPerArea[globalAreaCode] = [];
      }

      accountsPerArea[globalAreaCode].push({
        accountNumber,
        salesOrder,
        postCode,
        numbersRequired,
      });
    } catch (error) {
      provisioningListErrors.value.push(error.message);
    }
  }

  const date = new Date();

  const today = `${date.getFullYear()}-${
    date.getMonth() + 1
  }-${date.getDate()}`;

  const finalListInputElement = finalListInput.value;

  provisioningListOutput.value = "";
  finalListInputElement.value = "";

  for (const [globalAreaCode, lines] of Object.entries(accountsPerArea)) {
    provisioningListOutput.value += `${globalAreaCode} - ${areaCodeNames[globalAreaCode]}\n\n`;

    for (const line of lines) {
      for (const [, lineNumber] of line.numbersRequired) {
        provisioningListOutput.value += `,${line.accountNumber},${line.salesOrder},${today}\n`;
        finalListInputElement.value += `${line.accountNumber} ${line.salesOrder}  ${lineNumber} NEW\n`;
      }
    }

    provisioningListOutput.value += `\n`;
  }

  provisioningListOutput.value = provisioningListOutput.value.trim();
  finalListInputElement.value = finalListInputElement.value.trim();
}
</script>
