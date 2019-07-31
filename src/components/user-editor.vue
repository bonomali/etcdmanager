<template>
  <v-card>
    <v-toolbar dark flat>
      <v-toolbar-title>{{ title }}</v-toolbar-title>
      <v-divider class="mx-2" inset vertical></v-divider>
      <v-spacer></v-spacer>
    </v-toolbar>
    <v-container fill-height fluid>
      <v-layout fill-height>
        <v-flex xs12 align-end flexbox>
          <v-form ref="form" v-model="valid" lazy-validation>
            <v-text-field
              dark
              v-model="name"
              :error-messages="nameErrors"
              :label="$t('userEditor.fields.name.label')"
              :disabled="editMode"
              :placeholder="$t('userEditor.fields.name.placeholder')"
              required
              @input="$v.name.$touch()"
              @blur="$v.name.$touch()"
            >
              <v-tooltip slot="prepend" bottom max-width="200">
                <v-icon slot="activator" color="primary" dark>info</v-icon>
                <span>{{ $t('userEditor.fields.name.tooltip') }}</span>
              </v-tooltip>
            </v-text-field>

            <v-text-field
              dark
              v-model="password"
              :error-messages="passwordErrors"
              :label="$t('userEditor.fields.password.label')"
              :placeholder="$t('userEditor.fields.password.placeholder')"
              :type="getType()"
              :append-icon="showPassword ? 'visibility' : 'visibility_off'"
              counter
              @click:append="showPassword = !showPassword"
              @input="$v.password.$touch()"
              @blur="$v.password.$touch()"
            >
              <v-tooltip slot="prepend" bottom max-width="200">
                <v-icon slot="activator" color="primary" dark>info</v-icon>
                <span>{{ $t('userEditor.fields.password.tooltip') }}</span>
              </v-tooltip>
            </v-text-field>

            <v-text-field
              dark
              id="pwcheck"
              v-model="pwcheck"
              counter
              :label="$t('userEditor.fields.pwcheck.label')"
              :placeholder="$t('userEditor.fields.pwcheck.placeholder')"
              :type="getType()"
              required
            >
              <v-tooltip slot="prepend" bottom max-width="200">
                <v-icon slot="activator" color="primary" dark>info</v-icon>
                <span>{{ $t('userEditor.fields.pwcheck.tooltip') }}</span>
              </v-tooltip>
            </v-text-field>

            <template v-if="showRights">
            <h2 class="subheading">Roles</h2>
            <hr />
              <v-alert  v-if="roles.length === 0" :value="true" color="error" icon="warning">{{ $t('userEditor.messages.norights') }}</v-alert>
              <v-checkbox
                @change="setRole(role)"
                v-for="role in roles"
                v-model="ownRoles"
                v-bind:label="role.name"
                v-bind:key="role.name"
                v-bind:value="role.name"
              ></v-checkbox>
            </template>

            <v-btn :disabled="!valid" round color="primary" @click="submit">{{ opTitle }}</v-btn>
            <v-btn color="warning" round @click="cancel">{{ $t('common.actions.close.label') }}</v-btn>
            <v-spacer></v-spacer>
          </v-form>
        </v-flex>
      </v-layout>
    </v-container>
  </v-card>
</template>

<script lang='ts'>
import Component from 'vue-class-component';
import { Role } from 'etcd3';

import {
    required,
    alphaNum,
    sameAs,
    requiredIf,
} from 'vuelidate/lib/validators';
import Messages from '@/lib/messages';
import { BaseEditor } from '../lib/editor.class';
import { Prop } from 'vue-property-decorator';
import UserService from '../services/user.service';
import RoleService from '../services/role.service';
import store from '../store';
import { ValidationError } from '../lib/validation-error.class';

//@ts-ignore
class UserEditorError extends Error {
    constructor(message: any) {
        super(message);
        this.name = 'UserEditorError';
    }
}

@Component({
    // @ts-ignore
    name: 'key-editor',
    validations: {
        name: {
            required,
            alphaNum,
        },
        password: {
            requiredIf: requiredIf((nestedModel) => {
                return nestedModel.someFlag;
            }),
            sameAs: sameAs('pwcheck'),
            pwPattern: (value: string) => {
                const ptrn = store.state.users.pattern;
                return ptrn ? new RegExp(ptrn).test(value) : (
                    /^[^\s]{8,16}$/gi.test(value) &&
                    /[0-9]+/.test(value) &&
                    /[A-Z]+/.test(value)
                );
            },
        },
    },
})
export default class UserEditor extends BaseEditor {
    public itemType: string = 'user';
    public itemId: string = 'name';

    // @ts-ignore
    @Prop() data: {
        name: string;
        roles: Role[];
    };
    // @ts-ignore
    @Prop() mode: string;

    public name: string = this.data.name || '';
    public password: string = '';
    public pwcheck: string = '';
    public showPassword: boolean = false;
    public roles: Role[] = [];
    public ownRoles: string[] = [];
    public showRights: boolean = false;
    private roleService: RoleService;
    private userService: UserService;

    constructor() {
        super();
        this.roleService = new RoleService(
            this.$store.state.connection.getClient()
        );
        this.userService = new UserService(
            this.$store.state.connection.getClient()
        );
    }

    get nameErrors() {
        const errors: any = [];
        // @ts-ignore
        if (!this.$v.name.$dirty) {
            return errors;
        }
        // @ts-ignore
        if (!this.$v.name.required) {
            errors.push(this.$t('common.validation.required'));
        }
        // @ts-ignore
        if (!this.$v.name.alphaNum) {
            errors.push(this.$t('common.validation.alphanumeric'));
        }
        return errors;
    }

    get passwordErrors() {
        const errors: any = [];
        // @ts-ignore
        if (!this.$v.password.$dirty) {
            return errors;
        }
        // @ts-ignore
        if (!this.$v.password.sameAs) {
            errors.push(this.$t('userEditor.messages.pwmatch'));
        }
        // @ts-ignore
        if (!this.$v.password.pwPattern) {
            errors.push(this.$t('userEditor.messages.invalid'));
        }
        return errors;
    }

    public async setRole(
        role: Role
    ): Promise<UserEditor> {
        try {
            if (!this.ownRoles.includes(role.name)) {
                this.toggleLoading();
                await this.userService.revokeRole(
                    this.data.name,
                    role
                );
                this.toggleLoading();
            } else {
                this.toggleLoading();
                await this.userService.addRole(
                    this.data.name,
                    role
                );
                this.toggleLoading();
            }
            this.$store.commit('message', Messages.success());
            return Promise.resolve(this);
        } catch (error) {
            this.$store.commit('message', Messages.error(error));
            this.toggleLoading();
        }

        return Promise.resolve(this);
    }

    public async created() {
        try {
            this.roles = await this.roleService.getRoles();
            this.ownRoles = this.data.roles
                ? this.data.roles.map((role) =>  {
                    return role.name;
                })
                : [];
        } catch (error) {
            this.$store.commit('message', Messages.error(error));
        }

        this.showRights = !this.createMode;
    }

    public getType(): string {
        return this.showPassword ? 'text' : 'password';
    }

    public async submit(): Promise<UserEditor | ValidationError> {
        this.$v.$touch();
        if (this.$v.$invalid) {
            return Promise.reject(new ValidationError('Form is invalid..'));
        }

        try {
            this.toggleLoading();
            if (this.createMode) {
                await this.userService.createUser(this.name, this.password);
            } else {
                await this.userService.setPassword(this.name, this.password);
            }
            this.toggleLoading();
            this.$store.commit('message', Messages.success());
            this.$emit('reload');
            this.$v.$reset();
            if (this.createMode) {
                this.name = '';
                this.password = '';
                this.pwcheck = '';
                this.ownRoles = [];
                this.showRights = false;
            }
            return Promise.resolve(this);
        } catch (e) {
            this.$store.commit('message', Messages.error(e));
            this.toggleLoading();
        }

        return Promise.resolve(this);
    }
}
</script>

<style scoped>
</style>