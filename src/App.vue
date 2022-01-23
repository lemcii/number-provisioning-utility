<template>
  <div class="p-8 mx-auto max-w-xl">
    <h1 class="text-gray-50 text-3xl font-bold mb-4">
      Number Provisioning Utility
    </h1>
    <div class="mb-8">
      <label class="block mb-2 text-blue-300 font-semibold" for="input">
        Raw List
      </label>
      <prism-editor
        id="input"
        name="input"
        class="bg-gray-800 border border-gray-700 rounded px-4 py-3 w-full focus:outline-none focus:border-blue-500 hover:border-blue-500 cursor-text"
        @input="parseRawList($event.target.value)"
        @click="$event.target.getElementsByTagName('textarea')?.[0]?.focus()"
        v-model="code"
        :highlight="errorHighlight"
        insert-spaces="false"
        ignore-tab-key
        line-numbers
      />
    </div>
    <div class="mb-8">
      <div v-if="provisioningListErrors.length">
        <div
          class="text-red-200 bg-red-900/75 p-3 rounded border border-red-600 mb-5"
        >
          <h3 class="font-bold mb-2 text-white">
            {{ provisioningListErrors.length }} Error(s) Found
          </h3>
          <ul>
            <li v-for="error in provisioningListErrors">
              <span class="font-semibold text-red-100">
                Line {{ error[0] + 1 }}
              </span>
              &ndash; {{ error[1] }}
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
            <span
              :class="
                accounts.length ===
                accounts.filter(({ provisionedNumber }) => provisionedNumber)
                  .length
                  ? 'text-blue-400'
                  : 'text-red-500'
              "
            >
              {{
                accounts.filter(({ provisionedNumber }) => provisionedNumber)
                  .length
              }}/{{ accounts.length }}
            </span>
          </h3>
          <div class="space-y-3">
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
                :ref="`input-${globalAreaCode}`"
                class="bg-gray-800 border border-gray-700 rounded px-4 py-3 w-full h-32 focus:outline-none focus:border-blue-500 hover:border-blue-500"
                :class="
                  getInputLines(`input-${globalAreaCode}`) > 0 &&
                  accounts.length !== getInputLines(`input-${globalAreaCode}`)
                    ? 'text-red-100 border-red-900 hover:border-red-500 focus:border-red-500 bg-red-900/25'
                    : 'text-white'
                "
                @input="
                  updateLines($event.target.value, accounts, globalAreaCode)
                "
              />
              <div
                v-if="
                  getInputLines(`input-${globalAreaCode}`) &&
                  accounts.length !== getInputLines(`input-${globalAreaCode}`)
                "
                class="mt-1 mb-2 text-red-500"
              >
                Invalid line quantity
              </div>
            </div>
            <div>
              <div class="flex justify-between mb-2">
                <label
                  class="text-blue-300 font-semibold"
                  :for="`input-${globalAreaCode}`"
                >
                  Output
                </label>
                <button
                  class="text-xs font-bold h-6 px-1.5 rounded border border-blue-900 bg-blue-900/50 hover:border-blue-700 focus:text-blue-50 focus:bg-blue-600"
                  @click="
                    copy(
                      document.getElementById(`output-${globalAreaCode}`).value
                    )
                  "
                >
                  Copy
                </button>
              </div>
              <textarea
                :id="`output-${globalAreaCode}`"
                :name="`output-${globalAreaCode}`"
                class="bg-gray-800 border border-gray-700 rounded px-4 py-3 w-full h-32 focus:outline-none focus:border-blue-500 cursor-not-allowed"
                :value="formatLines(accounts)"
                readonly
              />
            </div>
          </div>
        </div>
      </div>
    </div>
    <div>
      <h2 class="text-xl mb-4 text-blue-50">
        <span class="font-bold">Number Provising Output</span> &ndash;
        <span
          :class="
            provisionedAccountTotal === accountTotal
              ? 'text-blue-400'
              : 'text-red-500'
          "
        >
          {{ provisionedAccountTotal }}/{{ accountTotal }}
        </span>
      </h2>
      <div class="flex justify-between mb-2">
        <label class="text-blue-300 font-semibold" for="input">
          Final List
        </label>
        <button
          class="text-xs font-bold h-6 px-1.5 rounded border border-blue-900 bg-blue-900/50 hover:border-blue-700 focus:text-blue-50 focus:bg-blue-600"
          @click="copy(document.getElementById('output').value)"
        >
          Copy
        </button>
      </div>
      <textarea
        id="output"
        name="output"
        class="bg-gray-800 border border-gray-700 rounded p-4 w-full h-64 focus:outline-none focus:border-blue-500 cursor-not-allowed"
        ref="finalListInput"
        readonly
      />
    </div>
  </div>
</template>

<script setup>
// this is bad, but who cares it's a temporary quick hacky tool

import { PrismEditor } from "vue-prism-editor";
import { ref } from "vue";
import sanitizeHtml from "sanitize-html";
import useClipboard from "vue-clipboard3";
import postcodeMapping from "./assets/postcodeMapping.json";

import "vue-prism-editor/dist/prismeditor.min.css";

/**
 * @typedef {{ accountNumber: string, salesOrder: string, lineNumber: string, provisionedNumber?: string }} Account
 */

const { toClipboard: copy } = useClipboard();

const date = new Date();
const today = `${date.getFullYear()}-${date.getMonth() + 1}-${date.getDate()}`;

const accountTotal = ref(0);
const provisionedAccountTotal = ref(0);

const document = window.document;

/** @type {import("vue").Ref<{ [area: string]: Account[] }>} */
const accountsByArea = ref({});

/** @type {import("vue").Ref<[number, string][]>} */
const provisioningListErrors = ref([]);

/** @type {import("vue").Ref<{ [area: string]: string }>} */
const globalAreaCodeNames = ref({});

/** @type {import("vue").Ref<Element>} */
const finalListInput = ref();

const regex =
  /^(\d+)\s+(\d+)\s+(?:Line\s+\d\s+Missing\s+){1,2}([a-z]{1,2}\d{1,2}\s\d[a-z]{2})$/i;

/**
 * @param {string} input
 */
function parseRawList(text) {
  /** @type {string[]} */
  const lines = text
    .split("\n")
    .map((value) => value.trim())
    .filter((value) => value);

  /** @type {{ [area: string]: Account[] }} */
  const accountsPerAreaCode = {};

  provisioningListErrors.value = [];

  for (const [index, line] of lines.entries()) {
    try {
      if (line.length === 0) {
        continue;
      }

      const match = line.match(regex);

      if (!match) {
        throw new Error(`Unable to parse "${line}"`);
      }

      const [, accountNumber, salesOrder, postCode] = match;

      const postCodeArea = postCode.match(/^([a-z0-9]+)/i);

      if (!postCodeArea) {
        throw new Error(`Order ${salesOrder} has invalid postcode`);
      }

      if (!postcodeMapping[postCodeArea[0]]) {
        throw new Error(
          `Order ${salesOrder} has unknown postcode area: ${postCodeArea[0]}`
        );
      }

      /** @type {[string, number]} */
      const [areaName, areaCode] = postcodeMapping[postCodeArea[0]];

      // TODO: check if numbers are correct, not e.g. line 1 and line 1 missing
      const numbersRequired = [...line.matchAll(/Line\s+(\d)\s+Missing/gi)];

      if (numbersRequired.length === 0 || numbersRequired.length > 2) {
        throw new Error(
          `Order ${salesOrder} has incorrect number of "line missing" values`
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
          lineNumber,
        });
      }
    } catch (error) {
      provisioningListErrors.value = [
        ...provisioningListErrors.value,
        [index, error.message],
      ];
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
 * @param {Account[]} accounts
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
  const accounts = Object.values(accountsByArea.value).flat();

  accountTotal.value = accounts.length;

  provisionedAccountTotal.value = accounts.filter(
    ({ provisionedNumber }) => provisionedNumber
  ).length;

  finalListInput.value.value = accounts
    .map(
      ({ accountNumber, salesOrder, provisionedNumber, lineNumber }) =>
        `${accountNumber} ${salesOrder} ${
          provisionedNumber || ""
        } ${lineNumber} NEW`
    )
    .join("\n");
}

/**
 * @param {string} code
 */
function errorHighlight(code) {
  return code
    .split("\n")
    .map((line, index) =>
      line.trim().length
        ? `<span class="${
            provisioningListErrors.value.find(
              ([errorIndex]) => index === errorIndex
            )
              ? "underline decoration-wavy decoration-red-500 bg-red-900/50"
              : "white"
          }">${sanitizeHtml(line)}</span>`
        : line
    )
    .join("\n");
}

/**
 * @param {string} id
 */
function getInputLines(id) {
  return (
    document
      .getElementById(id)
      ?.value?.split("\n")
      .filter((line) => line.trim().length).length || 0
  );
}
</script>
