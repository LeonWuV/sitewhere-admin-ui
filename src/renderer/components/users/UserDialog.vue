<template>
  <base-dialog :title="title" :width="width" :visible="dialogVisible"
    :createLabel="createLabel" :cancelLabel="cancelLabel" :error="error"
    @createClicked="onCreateClicked" @cancelClicked="onCancelClicked">
    <v-tabs dark v-model="active">
      <v-tabs-bar dark color="primary">
        <v-tabs-item key="details" href="#details">
          User Details
        </v-tabs-item>
        <v-tabs-item key="permissions" href="#permissions">
          Permissions
        </v-tabs-item>
        <v-tabs-item key="metadata" href="#metadata">
          Metadata
        </v-tabs-item>
        <v-tabs-slider></v-tabs-slider>
      </v-tabs-bar>
      <v-tabs-items>
        <v-tabs-content key="details" id="details">
          <v-card flat>
            <v-card-text>
              <v-container fluid>
                <v-layout row wrap>
                  <v-flex xs12>
                    <v-text-field required class="mt-1" label="Username"
                      v-model="userUsername" hide-details prepend-icon="info">
                    </v-text-field>
                    <div class="verror">
                      <span v-if="$v.userUsername.$invalid && $v.$dirty">Username is required.</span>
                    </div>
                  </v-flex>
                  <v-flex xs12>
                    <v-text-field type="password" required class="mt-1"
                      label="Password" v-model="userPassword"
                      hide-details prepend-icon="https">
                    </v-text-field>
                    <div class="verror">
                      <span v-if="!$v.userPassword.required && $v.$dirty">Password is required.</span>
                      <span v-if="!$v.userPassword.minLength && $v.$dirty">Password minimum lenght is 6.</span>
                    </div>
                  </v-flex>
                  <v-flex xs12>
                    <v-text-field type="password" required class="mt-1"
                      label="Password (confirm)" v-model="userPasswordConfirm"
                      hide-details prepend-icon="https">
                    </v-text-field>
                    <div class="verror">
                      <span v-if="!$v.userPasswordConfirm.sameAsPassword && $v.$dirty">Password does not match.</span>
                    </div>
                  </v-flex>
                  <v-flex xs12>
                    <v-text-field required class="mt-1" label="First name"
                      v-model="userFirstName" hide-details prepend-icon="info">
                    </v-text-field>
                    <div class="verror">
                      <span v-if="$v.userFirstName.$invalid && $v.$dirty">First name is required.</span>
                    </div>
                  </v-flex>
                  <v-flex xs12>
                    <v-text-field required class="mt-1" label="Last name"
                      v-model="userLastName" hide-details prepend-icon="info">
                    </v-text-field>
                    <div class="verror">
                      <span v-if="$v.userLastName.$invalid && $v.$dirty">Last name is required.</span>
                    </div>
                  </v-flex>
                  <v-flex xs12>
                    <v-select required :items="accountStatusList"
                      v-model="userAccountStatus" label="Account status"
                      prepend-icon="check_circle"></v-select>
                    <div class="verror">
                      <span v-if="$v.userAccountStatus.$invalid && $v.$dirty">Account status is required.</span>
                    </div>
                  </v-flex>
                </v-layout>
              </v-container>
            </v-card-text>
          </v-card>
        </v-tabs-content>
        <v-tabs-content class="user-permissions" key="permissions" id="permissions">
          <v-card flat>
            <v-card-text>
              <el-tree ref="tree" show-checkbox default-expand-all node-key="id"
                :data="allPermissions" :props="treeProps"
                :default-checked-keys="userAuthorities"
                @check-change="onPermissionsUpdated">
              </el-tree>
            </v-card-text>
          </v-card>
        </v-tabs-content>
        <v-tabs-content key="metadata" id="metadata">
          <metadata-panel :metadata="metadata"
            @itemDeleted="onMetadataDeleted" @itemAdded="onMetadataAdded"/>
        </v-tabs-content>
      </v-tabs-items>
    </v-tabs>
  </base-dialog>
</template>

<script>
import Utils from "../common/Utils"
import BaseDialog from "../common/BaseDialog"
import MetadataPanel from "../common/MetadataPanel"
import {_getAuthoritiesHierarchy} from "../../http/sitewhere-api-wrapper"
import { required, sameAs, minLength } from "vuelidate/lib/validators";

export default {

  data: () => ({
    active: null,
    menu: null,
    dialogVisible: false,
    userUsername: null,
    userPassword: null,
    userPasswordConfirm: null,
    userFirstName: null,
    userLastName: null,
    userAccountStatus: null,
    userAuthorities: [],
    metadata: [],
    treeProps: {
      children: "items",
      label: "text"
    },
    allPermissions: [],
    accountStatusList: [
      {
        "text": "Active",
        "value": "Active"
      }, {
        "text": "Expired",
        "value": "Expired"
      }, {
        "text": "Locked",
        "value": "Locked"
      }
    ],
    error: null
  }),

  validations: {
    userUsername: {
      required
    },
    userPassword: {
      required,
      minLength: minLength(6)
    },
    userPasswordConfirm: {
      sameAsPassword: sameAs('userPassword')
    },
    userFirstName: {
      required
    },
    userLastName: {
      required
    },
    userAccountStatus: {
      required
    }
  },

  components: {
    BaseDialog,
    MetadataPanel
  },

  props: ["title", "width", "createLabel", "cancelLabel"],

  methods: {
    // Generate payload from UI.
    generatePayload: function () {
      let payload = {};
      payload.username = this.$data.userUsername;
      payload.password = this.$data.userPassword;
      payload.firstName = this.$data.userFirstName;
      payload.lastName = this.$data.userLastName;
      payload.status = this.$data.userAccountStatus;
      payload.authorities = this.$data.userAuthorities;
      payload.metadata = Utils.arrayToMetadata(this.$data.metadata);
      return payload;
    },

    // Reset dialog contents.
    reset: function (e) {
      this.$data.userUsername = null;
      this.$data.userPassword = null;
      this.$data.userPasswordConfirm = null;
      this.$data.userFirstName = null;
      this.$data.userLastName = null;
      this.$data.userAccountStatus = null;
      this.$data.userAuthorities = [];
      this.$data.metadata = [];
      this.$data.active = "details";
      this.$v.$reset();

      // Reload permissions hierarchy.
      var component = this;
      _getAuthoritiesHierarchy(this.$store)
        .then(function (response) {
          component.allPermissions = response.data;
        }).catch(function (e) {
        })
    },

    // Load dialog from a given payload.
    load: function (payload) {
      this.reset();

      if (payload) {
        this.$data.userUsername = payload.username;
        this.$data.userFirstName = payload.firstName;
        this.$data.userLastName = payload.lastName;
        this.$data.userAccountStatus = payload.status;
        this.$data.userAuthorities = payload.authorities;
        this.$data.metadata = Utils.metadataToArray(payload.metadata);
      }
    },

    // Called to open the dialog.
    openDialog: function () {
      this.$data.dialogVisible = true;
    },

    // Called to open the dialog.
    closeDialog: function () {
      this.$data.dialogVisible = false;
    },

    // Called to show an error message.
    showError: function (error) {
      this.$data.error = error;
    },

    // Called after create button is clicked.
    onCreateClicked: function (e) {
      this.$v.$touch();
      if (this.$v.$invalid) {
        return;
      }
      var payload = this.generatePayload();
      this.$emit("payload", payload);
    },

    // Called after cancel button is clicked.
    onCancelClicked: function (e) {
      this.$data.dialogVisible = false;
    },

    // Called when permissions list is updated.
    onPermissionsUpdated: function () {
      this.$data.userAuthorities = this.$refs["tree"].getCheckedKeys();
    },

    // Called when a metadata entry has been deleted.
    onMetadataDeleted: function (name) {
      var metadata = this.$data.metadata;
      for (var i = 0; i < metadata.length; i++) {
        if (metadata[i].name === name) {
          metadata.splice(i, 1);
        }
      }
    },

    // Called when a metadata entry has been added.
    onMetadataAdded: function (entry) {
      var metadata = this.$data.metadata;
      metadata.push(entry);
    }
  }
}
</script>

<style scoped>
.user-permissions {
  max-height: 400px;
  overflow-y: auto;
}
</style>
