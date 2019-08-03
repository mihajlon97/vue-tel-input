<template>
  <div :class="['vue-tel-input', wrapperClasses, { disabled: disabled }]">
    <div
      v-click-outside="clickedOutside"
      :tabindex="dropdownOptions && dropdownOptions.tabindex ? dropdownOptions.tabindex : 0"
      :class="{ open: open }"
      class="dropdown"
      @keydown="keyboardNav"
      @click="toggleDropdown"
      @keydown.esc="reset"
    >
      <span class="selection">
        <div
          v-if="enabledFlags"
          :class="activeCountry.iso2.toLowerCase()"
          class="iti-flag" />
        <span
          v-if="enabledCountryCode"
          class="country-code">+{{ activeCountry.dialCode }}</span>
        <slot
          :open="open"
          name="arrow-icon">
          <span class="dropdown-arrow">{{ open ? "▲" : "▼" }}</span>
        </slot>
      </span>
	    <span v-if="open && this.searchCountries" style="max-width: 360px; border-bottom: 0px solid #ccc; z-index: 1; background-color: #f2f1f1; position:absolute; top: 30px; height: 30px; width: 80vw;">
		    <input style="padding-left: 20px;" type="text" ref="search" @focus="searchFocused = true" @blur="searchFocused = false" v-model="search" :placeholder="this.searchPlaceholder">
	    </span>
	    <ul
        v-show="open"
        ref="list">
	    <li
          v-for="(pb, index) in sortedCountries"
          :key="pb.iso2 + (pb.preferred ? '-preferred' : '')"
          :class="getItemClass(index, pb.iso2)"
          class="dropdown-item"
          @click="choose(pb)"
          @mousemove="selectedIndex = index"
        >
          <div
            v-if="enabledFlags"
            :class="pb.iso2.toLowerCase()"
            class="iti-flag" />
          <strong>{{ pb.name }}</strong>
          <span
            v-if="dropdownOptions && !dropdownOptions.disabledDialCode"
          >+{{ pb.dialCode }}</span
          >
        </li>
      </ul>
    </div>
    <input
      ref="input"
      v-model="phone"
      :placeholder="placeholder"
      :disabled="disabled"
      :required="required"
      :autocomplete="autocomplete"
      :name="name"
      :class="inputClasses"
      :id="inputId"
      :maxlength="maxLen"
      :tabindex="inputOptions && inputOptions.tabindex ? inputOptions.tabindex : 0"
      type="tel"
      @blur="onBlur"
      @input="onInput"
      @keyup.enter="onEnter"
      @keyup.space="onSpace"
    >
  </div>
</template>

<style src="./assets/sprite.css"></style>
<style scoped>
li.last-preferred {
  border-bottom: 1px solid #cacaca;
}
.iti-flag {
  margin-right: 5px;
  margin-left: 5px;
}
.dropdown-item .iti-flag {
  display: inline-block;
  margin-right: 5px;
}
.selection {
	margin-top: 5px;
	cursor: pointer;
	font-size: .8em;
	display: -webkit-box;
	display: -ms-flexbox;
	display: flex;
	-webkit-box-align: center;
	-ms-flex-align: center;
	align-items: center
}
.vue-tel-input {
	display: -webkit-box;
	display: -ms-flexbox;
	display: flex;
	text-align: left
}
.vue-tel-input:focus-within {
	border-bottom:solid 0px #5D9CEC!important;
	transition:all 250ms ease;
}
input {
	border: none;
	border-radius: 0 var(--border-radius) var(--border-radius) 0;
	width: 100%;
	outline: none;
	line-height:20px;
	border-bottom:solid 1px rgba(0,0,0,0.1);
	border-left:none!important;
	border-right:none!important;
	border-top:none!important;
	display:block;
	box-shadow: none;
	background-color:transparent;
}
ul {
	padding: 0;
	margin: 0;
	margin-top: 10px;
	margin-right: 30px;
	text-align: left;
	list-style: none;
	max-height: 200px;
	max-width: 360px;
	overflow-y: scroll;
	position: absolute;
	top: 50px;
	left: -1px;
	background-color: #f2f1f1;
	border: 1px solid #ccc;
	border-top: 0px solid #ccc;
	width: 80vw;
	z-index: 9999;
}
.dropdown {
	display: -webkit-box;
	display: -ms-flexbox;
	display: flex;
	-webkit-box-orient: vertical;
	-webkit-box-direction: normal;
	-ms-flex-direction: column;
	flex-direction: column;
	-ms-flex-line-pack: center;
	align-content: center;
	-webkit-box-pack: center;
	-ms-flex-pack: center;
	justify-content: center;
	position: relative;
	padding-top: 7px;
	padding-left: 0px;
	padding-bottom: 7px;
	padding-right: 7px;
}
.dropdown.open {
	background-color: #f3f3f3
}
.dropdown:hover {
  background-color: #f3f3f3;
}
.country-code {
  color: #666;
}
.dropdown-arrow {
	-webkit-transform: scaleY(.5);
	-ms-transform: scaleY(.5);
	transform: scaleY(.5);
	display: inline-block;
	color: #666
}
.dropdown-item {
	cursor: pointer;
	padding: 4px 15px;
	height: 40px;
}
.dropdown-item img{
	margin-top: 3px;
	float:left;
}
.dropdown-item strong{
	color: #4b4b4b;
	font-size: 13px!important;
	padding-right: 15px;
}
.dropdown-item span{
	color: #4b4b4b;
	font-size: 13px!important;
}
.dropdown-item[data-v-00867bb8]:hover {
	background-color: #f3f3f3
}
.dropdown-item.highlighted {
  background-color: #f3f3f3;
}
.dropdown-menu.show {
  max-height: 300px;
  overflow: scroll;
}
.vue-tel-input.disabled .selection,
.vue-tel-input.disabled .dropdown,
.vue-tel-input.disabled input {
  cursor: no-drop;
}
</style>

<script>
import {
  formatNumber, AsYouType, isValidNumber, parsePhoneNumberFromString,
} from 'libphonenumber-js';
import allCountries from './assets/all-countries';
import getCountry from './assets/default-country';

export default {
  name: 'VueTelInput',
  directives: {
    // Click-outside by BosNaufal: https://github.com/BosNaufal/vue-click-outside
    'click-outside': {
      bind(el, binding, vNode) {
        // Provided expression must evaluate to a function.
        if (typeof binding.value !== 'function') {
          const compName = vNode.context.name;
          let warn = `[Vue-click-outside:] provided expression ${binding.expression} is not a function, but has to be`;
          if (compName) {
            warn += `Found in component ${compName}`;
          }
          console.warn(warn);
        }
        // Define Handler and cache it on the element
        const { bubble } = binding.modifiers;
        const handler = (e) => {
          // Fall back to composedPath if e.path is undefined
          const path = e.path || (e.composedPath && e.composedPath());
          if (bubble || (path.length && !el.contains(path[0]) && el !== path[0])) {
            binding.value(e);
          }
        };
        el.__vueClickOutside__ = handler;
        // add Event Listeners
        document.addEventListener('click', handler);
      },
      unbind(el) {
        // Remove Event Listeners
        document.removeEventListener('click', el.__vueClickOutside__);
        el.__vueClickOutside__ = null;
      },
    },
  },
  props: {
    value: {
      type: String,
      default: '',
    },
    placeholder: {
      type: String,
      default: 'Enter a phone number',
    },
    disabledFetchingCountry: {
      type: Boolean,
      default: false,
    },
    disabled: {
      type: Boolean,
      default: false,
    },
    disabledFormatting: {
      type: Boolean,
      default: false,
    },
    invalidMsg: {
      default: '',
      type: String,
    },
    required: {
      type: Boolean,
      default: false,
    },
    allCountries: {
      type: Array,
      default: () => allCountries,
    },
    defaultCountry: {
      // Default country id, ie: 200 for Serbia
      // Will override the current country of user
      type: Number,
      default: '',
    },
    enabledCountryCode: {
      type: Boolean,
      default: false,
    },
    enabledFlags: {
      type: Boolean,
      default: true,
    },
    preferredCountries: {
      type: Array,
      default: () => [],
    },
    onlyCountries: {
      type: Array,
      default: () => [],
    },
    ignoredCountries: {
      type: Array,
      default: () => [],
    },
    autocomplete: {
      type: String,
      default: 'on',
    },
    name: {
      type: String,
      default: 'telephone',
    },
    wrapperClasses: {
      type: [String, Array, Object],
      default: '',
    },
    inputClasses: {
      type: [String, Array, Object],
      default: '',
    },
    inputId: {
      type: String,
      default: '',
    },
    dropdownOptions: {
      type: Object,
      default: () => ({}),
    },
    inputOptions: {
      type: Object,
      default: () => ({}),
    },
    maxLen: {
      type: Number,
      default: 25,
    },
    validCharactersOnly: {
      type: Boolean,
      default: false,
    },
    searchCountries: {
      type: Boolean,
      default: false,
    },
    searchPlaceholder: {
      type: String,
      default: 'Search...',
    },
  },
  data() {
    return {
      phone: '',
      search: '',
      activeCountry: { iso2: '' },
      open: false,
      searchFocused: false,
      selectedIndex: null,
      typeToFindInput: '',
      typeToFindTimer: null,
    };
  },
  computed: {
    mode() {
      if (!this.phone) {
        return '';
      }
      if (this.phone[0] === '+') {
        return 'code';
      }
      return 'normal';
    },
    filteredCountries() {
      // List countries after filtered
      if (this.onlyCountries.length) {
        return this.getCountries(this.onlyCountries);
      }

      if (this.ignoredCountries.length) {
        return this.allCountries.filter(({ iso2 }) => !this.ignoredCountries.includes(iso2.toUpperCase())
          && !this.ignoredCountries.includes(iso2.toLowerCase()));
      }

      return this.allCountries;
    },
    sortedCountries() {
      // Sort the list countries: from preferred countries to all countries
      const preferredCountries = this.getCountries(this.preferredCountries)
        .map(country => ({ ...country, preferred: true }));

      if (this.search === '')
        return [...preferredCountries, ...this.filteredCountries];
      else
        return [...preferredCountries, ...this.filteredCountries]
          .filter(c => c.name.toLowerCase().indexOf(this.search.toLowerCase()) !== -1 || c.dialCode.toLowerCase().indexOf(this.search.toLowerCase()) !== -1);
    },
    formattedResult() {
      // Calculate phone number based on mode
      if (!this.mode || !this.filteredCountries) {
        return '';
      }
      let { phone } = this;
      if (this.mode === 'code') {
        // If user manually type the country code
        const formatter = new AsYouType();// eslint-disable-line
        formatter.input(this.phone);

        // Find inputted country in the countries list
        this.activeCountry = this.findCountry(formatter.country) || this.activeCountry;
      } else if (this.mode === 'prefix') {
        // Remove the first '0' if this is a '0' prefix number
        // Ex: 0432421999
        phone = this.phone.slice(1);
      }
      if (this.disabledFormatting) {
        return this.phone;
      }

      return formatNumber(phone, this.activeCountry && this.activeCountry.iso2, 'International');
    },
    state() {
      return isValidNumber(this.formattedResult, this.activeCountry && this.activeCountry.iso2);
    },
    response() {
      // If it is a valid number, returns the formatted value
      // Otherwise returns what it is
      const response = {
        number: this.state ? this.formattedResult : this.phone,
        isValid: this.state,
        country: this.activeCountry,
      };
      // If formatting to the input is disabled, try to return the formatted value to its parent
      if (this.disabledFormatting) {
        Object.assign(response, {
          formattedNumber: formatNumber(this.phone, this.activeCountry && this.activeCountry.iso2, 'International'),
        });
      }
      return response;
    },
  },
  watch: {
    state(value) {
      if (value && this.mode !== 'prefix') {
        // If mode is 'prefix', keep the number as user typed,
        // Otherwise format it
        this.phone = this.formattedResult;
      }
      this.$emit('onValidate', this.response); // Deprecated
      this.$emit('validate', this.response);
    },
    value() {
      this.phone = this.value;
    },
    open(isDropdownOpened) {
      // Emit open and close events
      if (isDropdownOpened) {
        this.$emit('open');
        setTimeout(() => {
          this.$refs.search.focus();
        }, 500)
      } else {
        this.$emit('close');
      }
    },
    phone(newValue, oldValue) {
      if (this.validCharactersOnly && !this.testCharacters()) {
        this.$nextTick(() => { this.phone = oldValue; });
      }
    },
    activeCountry(value) {
      if (value && value.iso2) {
        this.$emit('country-changed', value);
      }
    }
  },
  mounted() {
    this.initializeCountry().then(() => {
      if (!this.phone
          && this.inputOptions
          && this.inputOptions.showDialCode
          && this.activeCountry && this.placeholder === '') {
        this.phone = `+${this.activeCountry.dialCode}`;
      }
      this.$emit('validate', this.response);
      this.$emit('onValidate', this.response); // Deprecated
    }).catch(console.error); // eslint-disable-line
  },
  created() {
    if (this.value) {
      this.phone = this.value.trim();
    }
  },
  methods: {
    initializeCountry() {
      return new Promise((resolve) => {
        /**
         * 1. If the phone included prefix (+12), try to get the country and set it
         */
        if (this.phone && this.phone[0] === '+') {
          const parsedPhone = parsePhoneNumberFromString(this.phone);
          if (parsedPhone && parsedPhone.country) {
            this.activeCountry = parsedPhone.country;
            resolve();
            return;
          }
        }
        /**
         * 2. Use default country if passed from parent
         */
        if (this.defaultCountry) {
          const defaultCountry = this.findCountry(this.defaultCountry, true);
          if (defaultCountry) {
            this.activeCountry = defaultCountry;
            resolve();
            return;
          }
        }
        /**
         * 3. Use the first country from preferred list (if available) or all countries list
         */
        this.activeCountry = this.findCountry(this.preferredCountries[0])
            || this.filteredCountries[0];
        /**
         * 4. Check if fetching country based on user's IP is allowed, set it as the default country
         */
        if (!this.disabledFetchingCountry) {
          getCountry().then((res) => {
            this.activeCountry = this.findCountry(res) || this.activeCountry;
          }).finally(resolve)
            .catch((error) => {
              console.warn(error);
            });
        } else {
          resolve();
        }
      });
    },
    /**
     * Get the list of countries from the list of iso2 code
     */
    getCountries(list = []) {
      return list
        .map(countryCode => this.findCountry(countryCode))
        .filter(Boolean);
    },
    findCountry(iso = '', findById = false) {
      return (!findById) ? this.allCountries.find(country => country.iso2 === iso.toUpperCase()) : this.allCountries.find(country => country.id === iso);
    },
    getItemClass(index, iso2) {
      const highlighted = this.selectedIndex === index;
      const lastPreferred = index === this.preferredCountries.length - 1;
      const preferred = this.preferredCountries.some(c => c.toUpperCase() === iso2);
      return {
        highlighted,
        'last-preferred': lastPreferred,
        preferred,
      };
    },
    choose(country) {
      this.activeCountry = country;
      if (this.inputOptions && this.inputOptions.showDialCode && country) {
        this.phone = `+${country.dialCode}`;
      }
      this.$emit('input', this.response.number, this.response);
      this.$emit('onInput', this.response); // Deprecated
      this.$refs.input.focus();
    },
    testCharacters() {
      const re = /^[()-+0-9\s]*$/;
      return re.test(this.phone) && (this.phone.match(/\+/g) || []).length <= 1;
    },
    onInput() {
      if (this.validCharactersOnly && !this.testCharacters()) {
        return;
      }
      this.$refs.input.setCustomValidity(this.response.isValid ? '' : this.invalidMsg);
      // Returns response.number to assign it to v-model (if being used)
      // Returns full response for cases @input is used
      // and parent wants to return the whole response.
      this.$emit('input', this.response.number, this.response);
      this.$emit('onInput', this.response); // Deprecated
    },
    onBlur() {
      this.$emit('blur');
      this.$emit('onBlur'); // Deprecated
    },
    onEnter() {
      this.$emit('enter');
      this.$emit('onEnter'); // Deprecated
    },
    onSpace() {
      this.$emit('space');
      this.$emit('onSpace'); // Deprecated
    },
    focus() {
      this.$refs.input.focus();
    },
    toggleDropdown() {
      if (this.disabled) {
        return;
      }
      if (this.searchFocused) {
        return;
      }
      this.open = !this.open;
    },
    clickedOutside() {
      this.open = false;
    },
    keyboardNav(e) {
      if (e.keyCode === 40) {
        // down arrow
        e.preventDefault();
        this.open = true;
        if (this.selectedIndex === null) {
          this.selectedIndex = 0;
        } else {
          this.selectedIndex = Math.min(this.sortedCountries.length - 1, this.selectedIndex + 1);
        }
        const selEle = this.$refs.list.children[this.selectedIndex];
        if (selEle.offsetTop + selEle.clientHeight
          > this.$refs.list.scrollTop + this.$refs.list.clientHeight) {
          this.$refs.list.scrollTop = selEle.offsetTop
            - this.$refs.list.clientHeight
            + selEle.clientHeight;
        }
      } else if (e.keyCode === 38) {
        // up arrow
        e.preventDefault();
        this.open = true;
        if (this.selectedIndex === null) {
          this.selectedIndex = this.sortedCountries.length - 1;
        } else {
          this.selectedIndex = Math.max(0, this.selectedIndex - 1);
        }
        const selEle = this.$refs.list.children[this.selectedIndex];
        if (selEle.offsetTop < this.$refs.list.scrollTop) {
          this.$refs.list.scrollTop = selEle.offsetTop;
        }
      } else if (e.keyCode === 13) {
        // enter key
        if (this.selectedIndex !== null) {
          this.choose(this.sortedCountries[this.selectedIndex]);
        }
        this.open = !this.open;
      } else {
        // typing a country's name
        this.typeToFindInput += e.key;
        clearTimeout(this.typeToFindTimer);
        this.typeToFindTimer = setTimeout(() => {
          this.typeToFindInput = '';
        }, 700);
        // don't include preferred countries so we jump to the right place in the alphabet
        const typedCountryI = this.sortedCountries
          .slice(this.preferredCountries.length)
          .findIndex(c => c.name.toLowerCase().startsWith(this.typeToFindInput));
        if (typedCountryI >= 0) {
          this.selectedIndex = this.preferredCountries.length + typedCountryI;
          const selEle = this.$refs.list.children[this.selectedIndex];
          const needToScrollTop = selEle.offsetTop < this.$refs.list.scrollTop;
          const needToScrollBottom = selEle.offsetTop + selEle.clientHeight
            > this.$refs.list.scrollTop + this.$refs.list.clientHeight;
          if (needToScrollTop || needToScrollBottom) {
            this.$refs.list.scrollTop = selEle.offsetTop - this.$refs.list.clientHeight / 2;
          }
        }
      }
    },
    reset() {
      this.selectedIndex = this.sortedCountries.map(c => c.iso2).indexOf(this.activeCountry.iso2);
      this.open = false;
    },
  },
};
</script>
