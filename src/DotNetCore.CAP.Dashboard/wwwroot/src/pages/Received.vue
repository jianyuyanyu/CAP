<template>
  <div>
    <b-row>
      <b-col md="3" class="mt-4">
        <b-list-group>
          <router-link class="list-group-item text-left list-group-item-secondary list-group-item-action"
            v-for="menu of subMens" :key="menu.text" active-class="active" :to="menu.name">
            {{ $t(menu.text) }}
            <b-badge :variant="menu.variant" class="float-right" pill> {{ onMetric[menu.num] }} </b-badge>
          </router-link>
        </b-list-group>
      </b-col>
      <b-col md="9">
        <h2 class="page-line mb-2">{{ $t("Received Message") }}</h2>
        <b-form class="d-flex">
          <div class="col-sm-10">
            <div class="form-row mb-2">
              <label class="sr-only" for="inline-form-input-name">{{ $t("Name") }}</label>
              <b-form-input v-model="formData.name" id="inline-form-input-name" class="form-control col mr-4"
                :placeholder="$t('Name')" />

              <label class="sr-only" for="inline-form-input-name">{{ $t("Group") }}</label>
              <b-form-input v-model="formData.group" id="inline-form-input-group" class="form-control col"
                :placeholder="$t('Group')" />
            </div>
            <div class="form-row">
              <label class="sr-only" for="inline-form-input-content">{{ $t("Content") }}</label>
              <b-form-input v-model="formData.content" id="inline-form-input-content" class="form-control"
                :placeholder="$t('Content')" />
            </div>
          </div>
          <b-button variant="dark" class="ml-2 align-self-end" @click="onSearch">
            <b-icon-search></b-icon-search>
            {{ $t("Search") }}
          </b-button>
        </b-form>
      </b-col>
    </b-row>
    <b-row>
      <b-col md="12">
        <b-btn-toolbar class="mt-4">
          <b-button size="sm" variant="dark" @click="reexecute" :disabled="!selectedItems.length" class="action-button">
            <b-icon-arrow-repeat aria-hidden="true"></b-icon-arrow-repeat>
            {{ $t("Re-execute") }}
          </b-button>
          <b-button size="sm" variant="danger" @click="deletemsg" :disabled="!selectedItems.length" class="action-button">
            <b-icon-trash aria-hidden="true"></b-icon-trash>
            {{ $t("Delete") }}
          </b-button>
          <div class="pagination">
            <span style="font-size: 14px"> {{ $t("Page Size") }}:</span>
            <b-button-group class="ml-2">
              <b-button variant="outline-secondary" size="sm" v-for="size in pageOptions"
                :class="{ active: formData.perPage == size }" @click="pageSizeChange(size)" :key="size">{{ size }}
              </b-button>
            </b-button-group>
          </div>
        </b-btn-toolbar>
        <b-table id="datatable" class="mt-3" :busy="isBusy" striped thead-tr-class="text-left" head-variant="light"
          tbody-tr-class="text-left" small :fields="fields" :items="items" select-mode="range">
          <template #table-busy>
            <div class="text-center text-secondary my-2">
              <b-spinner class="align-middle"></b-spinner>
              <strong class="ml-2">{{ $t("Loading") }}...</strong>
            </div>
          </template>

          <template #head(checkbox)="">
            <b-form-checkbox @change="selectAll" v-model="isSelectedAll"></b-form-checkbox>
          </template>

          <template #cell(checkbox)="data">
            <b-form-checkbox v-model="data.item.selected" @change="select(data.item)">
            </b-form-checkbox>
          </template>

          <template #cell(id)="data">
            <b-link @click="info(data.item, $event.target)">
              {{ data.item.id }}
            </b-link><br />
            {{ data.item.name }}
          </template>

          <template #cell(group)="data">
            <span class="text-break"> {{ data.item.group }}</span>
          </template>

        </b-table>
        <span class="float-left"> {{ $t("Total") }}: {{ totals }} </span>
        <b-pagination :first-text="$t('First')" :prev-text="$t('Prev')" :next-text="$t('Next')" :last-text="$t('Last')"
          v-model="formData.currentPage" :total-rows="totals" :per-page="formData.perPage" class="capPagination"
          aria-controls="datatable"></b-pagination>
      </b-col>
    </b-row>
    <b-modal size="lg" :id="infoModal.id" :title="'Id: ' + infoModal.title" ok-only ok-variant="secondary">
      <vue-json-pretty showSelectController :key="infoModal.id" :data="infoModal.content" />
    </b-modal>
  </div>
</template>
<script>
import axios from "axios";
import JSONBIG from "json-bigint";
import {
  BIconArrowRepeat,
  BIconSearch,
    BIconTrash,
} from 'bootstrap-vue';

const formDataTpl = {
  currentPage: 1,
  perPage: 10,
  name: "",
  group: "",
  content: "",
};
export default {
  components: {
    BIconArrowRepeat,
    BIconSearch,
    BIconTrash
  },
  props: {
    status: {},
  },
  data() {
    return {
      subMens: [
        {
          variant: "secondary",
          text: "Succeeded",
          num: 'receivedSucceeded',
          name: "/received/succeeded",
        },
        {
          variant: "danger",
          text: "Failed",
          name: "/received/failed",
          num: 'receivedFailed',
        },
      ],
      pageOptions: [10, 20, 50, 100, 500],
      selectedItems: [],
      isBusy: false,
      tableValues: [],
      isSelectedAll: false,
      formData: { ...formDataTpl },
      totals: 0,
      items: [],
      infoModal: {
        id: "info-modal",
        title: "",
        content: "{}",
      },
    };
  },
  computed: {
    onMetric() {
      return this.$store.getters.getMetric;
    },
    fields() {
      return [
        { key: "checkbox", label: "" },
        { key: "id", label: this.$t("IdName") },
        { key: "group", label: this.$t("Group") },
        { key: "retries", label: this.$t("Retries") },
        {
          key: "added",
          label: this.$t("Added"),
          formatter: (val) => {
            if (val != null) return new Date(val).format("yyyy-MM-dd hh:mm:ss");
          },
        },
        {
          key: "expiresAt",
          label: this.$t("Expires"),
          formatter: (val) => {
            if (val != null) return new Date(val).format("yyyy-MM-dd hh:mm:ss");
          },
        },
      ]
    }
  },
  mounted() {
    this.fetchData();
  },
  watch: {
    status: function () {
      this.fetchData();
    },
    "formData.currentPage": function () {
      this.fetchData();
    },
  },
  methods: {
    fetchData() {
      this.isBusy = true;
      axios.get(`/received/${this.status}`, {
        params: this.formData
      }).then(res => {
        this.items = res.data.items;
        this.totals = res.data.totals;
      }).finally(() => {
        this.isBusy = false;
      });
    },
    selectAll(checked) {
      if (checked) {
        this.selectedItems = [
          ...this.items.map((item) => {
            return {
              ...item,
              selected: true,
            };
          }),
        ];
        this.items = [...this.selectedItems];
      } else {
        this.selectedItems = [];
        this.items = this.items.map((item) => {
          return {
            ...item,
            selected: false,
          };
        });
      }
    },
    select(item) {
      const { id } = item;
      if (!this.selectedItems.some((item) => item.id == id)) {
        this.selectedItems.push(item);
      } else {
        this.selectedItems = this.selectedItems.filter((item) => item.id != id);
      }
      this.isSelectedAll = this.selectedItems.length == this.items.length;
    },
    clearSelected() {
      this.allSelected = false;
      this.selectedItems = [];
    },
    info(item, button) {
      this.infoModal.title = item.id.toString();
      this.infoModal.content = JSONBIG({ storeAsString: true }).parse(item.content.trim());
      this.$root.$emit("bv::show::modal", this.infoModal.id, button);
    },
    pageSizeChange: function (size) {
      this.formData.perPage = size;
      this.fetchData();
    },
    onSearch: function () {
      this.fetchData();
    },
    reexecute: function () {
      const _this = this;
      axios.post('/received/reexecute', this.selectedItems.map((item) => item.id)).then(() => {
        this.selectedItems.map((item) => {
          _this.$bvToast.toast(this.$t("ReexecuteSuccess") + "   " + item.id, {
            title: "Tips",
            variant: "secondary",
            autoHideDelay: 1000,
            appendToast: true,
            solid: true
          });
        });
        _this.fetchData();
        _this.clear();
      });
    },
    deletemsg: function () {
      const _this = this;
      axios.post('/received/delete', this.selectedItems.map((item) => item.id)).then(() => {
        this.selectedItems.map((item) => {
          _this.$bvToast.toast(this.$t("DeleteSuccess") + "   " + item.id, {
            title: "Tips",
            variant: "secondary",
            autoHideDelay: 1000,
            appendToast: true,
            solid: true
          });
        });
        _this.fetchData();
        _this.clear();
      });
    },
    clear() {
      this.items = this.items.map((item) => {
        return {
          ...item,
          selected: false,
        };
      });
      this.selectedItems = [];
      this.isSelectedAll = false;
    },
  },
};
</script>

<style scoped>
.pagination {
  flex: 1;
  justify-content: flex-end;
  align-items: center;
}

.capPagination::v-deep .page-link {
  color: #6c757d;
  box-shadow: none;
  border-color: #6c757d;
}

.capPagination::v-deep .page-link:hover {
  color: #fff;
  background-color: #6c757d;
  border-color: #6c757d;
}

.capPagination::v-deep .active .page-link {
  color: white;
  background-color: black;
}

.action-button {
  margin-right: 1rem;
}
</style>