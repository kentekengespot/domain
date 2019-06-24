<template>
  <v-form v-model="valid" ref="form" id="form" lazy-validation>
    <div v-for="category in filters" v-bind:key="category.title" class="category">
      <h2 class="headline">
        <v-icon v-if="filters.indexOf(category) == 0" @click="$router.go(-1)">keyboard_arrow_left</v-icon>
        {{category.title}}
        <v-icon v-if="filters.indexOf(category) != 0" @click="slide">keyboard_arrow_down</v-icon>
      </h2>
      <v-layout row wrap v-if="filters.indexOf(category) === 0">
        <v-flex xs12 pa-2>
          <input
            type="file"
            ref="fileInput"
            accept="image/png, image/jpeg"
            multiple
            @change="selectFile"
          >
          <v-layout row wrap>
            <v-flex xs3 v-for="(image) in previewImages" :key="'image' + image.index">
              <v-card>
                <v-card-text>
                  <v-img :src="image.src"/>
                </v-card-text>
                <v-card-actions>
                  <v-spacer/>
                  <v-icon @click="removeFile(image.index)">delete</v-icon>
                </v-card-actions>
              </v-card>
            </v-flex>
          </v-layout>
        </v-flex>
        <v-flex xs12 pa-2>
          <v-text-field
            v-model="filters[0].searchFilters[0].value"
            @input="addInput('LIC', $event)"
            @change="getLicenseData($event)"
            clearable
            label="Kenteken"
          ></v-text-field>
        </v-flex>
      </v-layout>
      <v-layout row wrap v-bind:key="refresh">
        <template v-for="filter in category.searchFilters">
          <v-flex v-if="filter.type === 'MakeModel'" v-bind:key="filter.header" xs12>
            <ModelMakeCreate
              ref="modelMake"
              v-on:update="addMakeModel"
              v-bind:item="filter"
              v-bind:makeInit="make"
              v-bind:modelInit="model"
            />
          </v-flex>
          <v-flex v-else-if="filter.type === 'Checkbox'" v-bind:key="filter.header" xs12 md6 pa-2>
            <CheckboxWrapper v-on:update="addInput" v-bind:item="filter"/>
          </v-flex>
          <v-layout
            row
            wrap
            v-else-if="filter.type === 'CheckboxList'"
            v-bind:key="filter.header"
            pa-2
          >
            <v-flex xs12 md3 v-for="checkbox in filter.options" v-bind:key="checkbox.value">
              <CheckboxList
                v-bind:item="filter"
                v-bind:checkbox="checkbox"
                v-on:update="addCheckbox"
              />
            </v-flex>
          </v-layout>
          <v-flex v-else-if="filter.type === 'Number'" v-bind:key="filter.header" xs12 md6 pa-2>
            <v-text-field
              v-model="filter.value"
              :rules="filter.required ? rules : []"
              type="number"
              min="1"
              @input="addInput(filter.input, $event)"
              :value="filter.value"
              clearable
              :required="filter.required"
              :class="filter.required ? 'required' : null"
              :label="filter.header"
            ></v-text-field>
          </v-flex>
          <v-flex v-else-if="filter.type === 'Text'" v-bind:key="filter.header" xs12 md6 pa-2>
            <v-text-field
              v-model="filter.value"
              :rules="filter.required ? rules : []"
              @input="addInput(filter.input, $event)"
              :value="filter.value"
              clearable
              :required="filter.required"
              :class="filter.required ? 'required' : null"
              :label="filter.header"
            ></v-text-field>
          </v-flex>
          <v-flex v-else-if="filter.type === 'BigText'" v-bind:key="filter.header" xs12 md6 pa-2>
            <v-textarea
              v-model="filter.value"
              :rules="filter.required ? rules : []"
              @input="addInput(filter.input, $event)"
              :value="filter.value"
              clearable
              :required="filter.required"
              :class="filter.required ? 'required' : null"
              :label="filter.header"
            ></v-textarea>
          </v-flex>
          <v-flex v-else-if="filter.type === 'DropDown'" v-bind:key="filter.header" xs12 md6 pa-2>
            <SelectWrapper v-on:update="addInput" :item="filter" clearable/>
          </v-flex>
          <v-flex
            v-else-if="filter.type === 'ColorDropDown'"
            v-bind:key="filter.header"
            xs12
            md6
            pa-2
          >
            <ColorDropDown v-on:update="addInput" :item="filter" clearable/>
          </v-flex>
          <v-flex v-else-if="filter.type === 'TextList'" v-bind:key="filter.header" xs12 md6 pa-2>
            <TextList v-on:update="addInput" :item="filter" clearable/>
          </v-flex>
          <v-flex v-else-if="filter.type === 'Date'" v-bind:key="filter.header" xs12 md6 pa-2>
            <DatePickerWrapper v-on:update="addInput" :item="filter" clearable/>
          </v-flex>
        </template>
      </v-layout>
    </div>
    <v-layout>
      <v-spacer/>
      <v-btn color="error" @click="clear">
        Legen
        &nbsp;
        <v-icon>clear</v-icon>
      </v-btn>
      <v-btn color="primary" @click="update" :disabled="!valid">
        Aanpassen
        &nbsp;
        <v-icon>add</v-icon>
      </v-btn>
    </v-layout>
  </v-form>
</template>

<script>
import ModelMakeCreate from "./Inputs/ModelMakeCreate.vue";
import TextList from "./Inputs/TextList.vue";
import SelectWrapper from "./Inputs/SelectWrapper.vue";
import CheckboxWrapper from "./Inputs/CheckboxWrapper.vue";
import CheckboxList from "./Inputs/CheckboxList.vue";
import Vue from "vue";
import DatePickerWrapper from "./Inputs/DatePickerWrapper.vue";
import ColorDropDown from "./Inputs/ColorDropDown.vue";
import bearer from "../../../mixins/bearer.vue";
import { constants } from "crypto";

export default {
  name: "edit-listing-form",
  components: {
    ModelMakeCreate,
    TextList,
    SelectWrapper,
    CheckboxWrapper,
    CheckboxList,
    Vue,
    DatePickerWrapper,
    ColorDropDown
  },
  data() {
    return {
      filters: [],
      valid: false,
      searchKeys: [],
      refresh: false,
      previewImages: [],
      previewImagesIndex: 0,
      make: null,
      model: null,
      listingId: null,
      rules: [v => !!v || "Dit veld is vereist"]
    };
  },
  mixins: [bearer],
  created() {
    axios
      .get("/vehiclelisting/listingoptions", this.getBearer())
      .then(response => {
        this.filters = response.data.searchForms;
        this.getListingData();
      })
      .catch(err => console.log(err));
  },
  methods: {
    getListingData() {
      axios
        .get(
          "/vehiclelisting/userlistings/edit/" + this.$route.params.id,
          this.getBearer()
        )
        .then(response => {
          this.listingId = this.$route.params.id;
          console.log(response.data);
          this.fillInForm(new URLSearchParams(response.data));
        })
        .catch(err => console.log(err));
    },
    getLicenseData(license) {
      this.clear();
      Vue.nextTick().then(() => {
        this.filters[0].searchFilters[0].value = license;
        this.addInput("LIC", license);
      });
      axios
        .get("/rdw/newlisting/" + license)
        .then(response => {
          this.fillInForm(new URLSearchParams(response.data));
        })
        .catch(err => console.log(err));
    },
    fillInForm(params) {
      params.forEach((value, key) => {
        for (var i in this.filters) {
          var category = this.filters[i];
          for (var j in category.searchFilters) {
            var item = category.searchFilters[j];

            if (item.input === key) {
              if (item.input === "MK") {
                this.make = parseInt(value);
                this.model = parseInt(params.get("MD"));
                this.addMakeModel(this.make, this.model);
              } else if (
                item.input === "HO" ||
                item.input === "EQ" ||
                item.input === "VPP" ||
                item.input === "VNP"
              ) {
                item.value = value;
              } else {
                item.value = value;
                this.addInput(key, value);
              }
            }
          }
        }
      });
      this.refresh = !this.refresh;
    },
    clear() {
      this.$refs.form.reset();
      this.searchKeys = [];

      for (var category of this.filters) {
        for (var filter of category.searchFilters) {
          filter.value = null;
        }
      }
    },
    addMakeModel(makeId, modelId) {
      if (this.searchKeys.length > 0) {
        this.searchKeys = this.searchKeys.filter(
          x => x.key != "MK" || x.key != "MD"
        );
      }
      if (makeId != null && makeId != false) {
        this.searchKeys.push({ key: "MK", value: makeId });

        if (modelId != null && modelId != null) {
          this.searchKeys.push({ key: "MD", value: modelId });
        }
      }
    },
    addInput(key, value) {
      if (this.searchKeys.length > 0) {
        this.searchKeys = this.searchKeys.filter(x => x.key != key);
      }
      if (value != null && value != false) {
        this.searchKeys.push({ key, value });
      }
    },
    addCheckbox(key, value, id) {
      var existingKey = this.searchKeys.find(x => x.key == key);
      if (existingKey) {
        var list = existingKey.value.toString().split(",");
        if (value == false || value == null) {
          list = list.filter(x => x != id);
        } else {
          if (!list.includes(id.toString())) {
            list.push(id);
          }
        }
        existingKey.value = list.join();
        if (existingKey.value == "") {
          this.searchKeys = this.searchKeys.filter(x => x.key != key);
        }
      } else {
        this.searchKeys.push({ key, value: id });
      }
    },
    createListingObject() {
      var listingQuery =
        "{" +
        this.searchKeys
          .map(x => '"' + x.key + '"' + ":" + '"' + x.value + '"')
          .join(",") +
        "}";

      var listing = JSON.parse(listingQuery);
      if (listing.HO != null) {
        listing.HO = this.toIEnumerable(listing.HO.split(","), true);
      }
      if (listing.EQ != null) {
        listing.EQ = this.toIEnumerable(listing.EQ.split(","), true);
      }
      if (listing.VPP != null) {
        listing.VPP = this.toIEnumerable(listing.VPP.split(","), false);
      }
      if (listing.VNP != null) {
        listing.VNP = this.toIEnumerable(listing.VNP.split(","), false);
      }
      return listing;
    },
    toIEnumerable(toConvert, int) {
      var converted = [];

      for (var option of toConvert) {
        if (int) {
          converted.push(parseInt(option));
        } else {
          converted.push(option);
        }
      }
      return converted;
    },
    update() {
      if (this.$refs.form.validate()) {
        var updatedListing = this.createListingObject();
        console.log(updatedListing);
        axios
          .put(
            "/vehiclelisting/" + this.listingId,
            updatedListing,
            this.getBearer()
          )
          .then(response => {
            console.log(response);
            this.$router.push(`/zoeken/voertuig/${this.listingId}`);
          })
          .catch(error => {
            console.log(error.response.data);
            this.$store.commit("showSnackbar", error.response.data);
          });
      }
    },
    slide(event) {
      var on = event.target.parentNode.parentNode.classList.toggle("show");
      event.target.innerHTML = on ? "keyboard_arrow_up" : "keyboard_arrow_down";
    },
    selectFile(input) {
      console.log(input);
      var self = this;
      if (input.target.files) {
        for (var file of input.target.files) {
          var reader = new FileReader();
          reader.onload = function(e) {
            self.previewImages.push({
              index: self.previewImagesIndex++,
              src: e.target.result
            });
          };
          reader.readAsDataURL(file);
        }
      }
    },
    removeFile(index) {
      this.previewImages = this.previewImages.filter(x => x.index != index);
    }
  }
};
</script>

<style>
#form {
  text-overflow: ellipsis;
}

.show .layout {
  display: flex !important;
}

.category:not(:first-child) .layout {
  display: none;
}

.required label::after {
  content: " *";
  color: red;
}
</style>