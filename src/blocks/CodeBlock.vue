<template>
  <div class="codeBlockContainer" data-aos="fade-right" data-aos-delay="100" data-aos-duration="1200">
    <div class="codeBlockTop">
      <div class="codeBlockHeader">
        <div class="codeBlockCircles">
          <div class="red"></div>
          <div class="yellow"></div>
          <div class="green"></div>
        </div>
        <div class="fileTab codeText" :class="{'error': !!error}">filename.{{ transpiled ? 'zinc' : 'sol' }}</div>
        <div class="errorText">{{ error }}</div>
      </div>
      <transition name="fade">
        <div v-if="opened===false || loading===true" class="overlay">
          <img v-if="!loading" alt="By developers, for developers" src="@/assets/images/pages/index/code.jpg">
          <div v-else class="loaderContainer">
            <loader/>
          </div>
        </div>
      </transition>
      <prism-editor v-if="!transpiled" v-model="code" class="editor" :highlight="highlighter" line-numbers/>
      <prism-editor v-else v-model="transpiled" class="editor" readonly :highlight="rustHighlighter" line-numbers/>
    </div>
    <div class="codeBlockFooter">
      <div class="buttonBlock">
<!--        <i-button v-if="opened===false" class="_padding-x-2" size="lg" variant="secondary" @click="opened=true">-->
<!--          Try our transpiler-->
<!--        </i-button>-->
        <i-button v-if="opened===false" class="_padding-x-2 antilink" size="lg" variant="secondary" :href="'/dev'" target="_blank">
          Get started!
        </i-button>
        <div v-else-if="transpiled===false" class="buttonGrid">
          <i-button class="_padding-x-2" size="lg" variant="secondary"
                    :disabled="!code || code.length<=0 || loading" @click="transpile()">
            Transpile!
          </i-button>
          <i-button size="lg" class="fal fa-times-circle _text-white _padding-x-2" @click="transpiled=false;opened=false;">
          </i-button>
        </div>
        <div v-else class="buttonGrid">
          <i-tooltip trigger="click">
            <i-button class="_padding-x-2" size="lg" variant="transparent" @click="copy()">
              <em class="fal fa-copy _text-white"></em>
            </i-button>
            <template slot="body">Copied!</template>
          </i-tooltip>
          <i-button class="_padding-x-2" size="lg" variant="transparent" @click="transpiled=false">
            <em class="fal fa-pen-alt _text-white"/>
          </i-button>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from "vue";
/**
 * @var document
 */
// @ts-ignore: Unreachable code error
import httpinvoke from "httpinvoke";
import { PrismEditor } from "vue-prism-editor";
// @ts-ignore: Unreachable code error
import { highlight, languages } from "prismjs/components/prism-core";

import "prismjs/components/prism-clike";
import "prismjs/components/prism-rust";
import "prismjs/components/prism-solidity";

export default Vue.extend({
  components: {
    PrismEditor,
  },
  data: () => ({
    code: `// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0;

struct Funder {
    address addr;
    uint amount;
}

contract CrowdFunding {
    struct Campaign {
        address payable beneficiary;
        uint fundingGoal;
        uint numFunders;
        uint amount;
    }

    uint numCampaigns;
    mapping (uint => Campaign) campaigns;

    function newCampaign(address payable beneficiary, uint goal) public returns (uint campaignID) {
        campaignID = numCampaigns++;
        Campaign storage c = campaigns[campaignID];
        c.beneficiary = beneficiary;
        c.fundingGoal = goal;
    }

    function contribute(uint campaignID) public payable {
        Campaign storage c = campaigns[campaignID];
        c.amount += msg.value;
    }

    function checkGoalReached(uint campaignID) public view returns (bool reached) {
        Campaign storage c = campaigns[campaignID];
        if (c.amount < c.fundingGoal)
            return false;
        return true;
    }
}
`,
    error: "",
    loading: false,
    opened: false,
    transpiled: false as false | string,
  }),
  methods: {
    highlighter(code: string) {
      return highlight(code, languages.solidity, languages.solidity);
    },
    rustHighlighter(code: string) {
      return highlight(code, languages.rust, languages.rust);
    },
    async transpile() {
      this.loading = true;
      this.error = "";
      try {
        const response = await httpinvoke("https://sol2zinc.zksync.dev/api/v1/transpiler", "POST", {
          input: this.code,
        });
        // eslint-disable-next-line no-prototype-builtins
        if (typeof response !== "object" || !response.hasOwnProperty("body")) {
          throw new Error("Code error");
        }
        this.transpiled = response.body;
      } catch (error) {
        this.error = error.message;
      }
      this.loading = false;
    },
    copy() {
      if (process.client && document) {
        const elem = document.createElement("textarea");
        elem.style.position = "absolute";
        elem.style.left = -99999999 + "px";
        elem.style.top = -99999999 + "px";
        elem.value = this.transpiled as string;
        document.body.appendChild(elem);
        elem.select();
        // noinspection JSUnresolvedFunction
        document.execCommand("copy");
        document.body.removeChild(elem);
      }
    },
  },
});
</script>
