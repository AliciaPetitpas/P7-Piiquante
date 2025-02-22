<template>    
<div>
    <MenuPage/>

<main id="main" class="main">

    <div class="profile">
        <p class="msg">{{ error }}</p>
        <p class="msg">{{ success }}</p>
        
        <!-- User Info -->
        <div class="user">
            <p class="text-admin" v-if="userInfo.admin">Ce compte appartient à un chargé de communication</p>
            <p class="text-admin" v-else>Ce compte appartient à un employé</p>
            <div v-if="!userInfo.admin">
                <input v-model="state.input.passwordadmin" type="password" class="input-form" placeholder="Mot de passe admin"/>
                <span v-if="v$admin.input.passwordadmin.$error" class="error">
                    {{ v$admin.input.passwordadmin.$errors[0].$message }}
                </span>
                <button @click="goAdmin()" class="user-admin">Administrateur</button>
            </div>
            
            <div class="user-profile-picture">
                <img :src="userInfo.imageUrl" ref="photoProfil" alt="Photo de profil" class="user-picture">
                <img class="user-picture" style="display:none" ref="filePreview" src="" alt="">
                
                <input 
                    style="display: none"
                    type="file" 
                    accept=".png, .jpg, .jpeg"
                    @change="onFileSelected"
                    ref="fileInput">

                <p class="text-profile-picture">Pour ajouter une photo de profil cliquez d'abord sur "choisir une image" pour sélectionner votre photo, ensuite cliquez sur "importer" pour la valider</p>
                <button @click="$refs.fileInput.click()" class="add-file">Choisir une image</button>
                <button @click="onUpload()" class="add-img">Importer</button>
            </div>

            <!-- Inputs modifications -->
            <p class="infos">Informations utilisateur :</p>
            <input v-model="state.input.newlast_name" type="text" class="input-form" placeholder="Nom"/>
            <span v-if="v$.input.newlast_name.$error" class="error">
                {{ v$.input.newlast_name.$errors[0].$message }}
            </span>
            <input v-model="state.input.newfirst_name" type="text" class="input-form" placeholder="Prénom"/>
            <span v-if="v$.input.newfirst_name.$error" class="error">
                {{ v$.input.newfirst_name.$errors[0].$message }}
            </span>
            <input v-model="state.input.newemail" type="email" class="input-form" placeholder="Adresse mail"/>
            <span v-if="v$.input.newemail.$error" class="error">
                {{ v$.input.newemail.$errors[0].$message }}
            </span>
            
                <!-- Bouton modification profil -->
            <button @click="modifyUser()" class="modify-user-info">Modifier informations</button>

            <button @click="modifyPassword()" class="modify-password">Changer le mot de passe ici</button>
            
            <button @click="deactivate()" class="deactivate">Désactiver mon compte</button>

        </div>
    </div>

</main>
</div>

</template>

<script>

import MenuPage from '../components/MenuPage.vue';
import { mapState } from 'vuex';
import useValidate from '@vuelidate/core';
import { required, minLength, helpers, email } from '@vuelidate/validators';
import { reactive, computed } from "vue";

export default {
    name: 'UserProfile',
    components: {
        MenuPage,
    },
    setup() {
        const state = reactive({
            input: {
                newlast_name: "",
                newfirst_name: "",
                newemail: "",
                passwordadmin: "",
            }
        });
        const rules = computed(() => {
            return {
                input: {
                    newlast_name : {
                        required: helpers.withMessage(
                            "Veuillez renseigner ce champ",
                            required
                        ),
                        minLength: helpers.withMessage(
                            "Le nom de peut comporter qu'une seule lettre",
                            minLength(2)
                        ),
                    },
                    newfirst_name: {
                        required: helpers.withMessage(
                            "Veuillez renseigner ce champ",
                            required
                        ),
                        minLength: helpers.withMessage(
                            "Le prénom de peut comporter qu'une seule lettre",
                            minLength(2)
                        ),
                    },
                    newemail: {
                        required: helpers.withMessage(
                            "Veuillez renseigner ce champ",
                            required
                        ),
                        email: helpers.withMessage(
                            "Veuillez saisir une adresse mail valide",
                            email
                        ),
                    },
                }
                
            }
        });
        const rulesAdmin = computed(() => {
            return {
                input: {
                    passwordadmin: {
                        required: helpers.withMessage(
                            "Veuillez renseigner ce champ",
                            required
                        ),
                    },
                }
                
            }
        });
        const v$ = useValidate(rules, state);
        const v$admin = useValidate(rulesAdmin, state);
        return {
            state,
            v$,
            v$admin,
        };
    },
    data: function() {
        return {
            selectedFile: null,
            error: "",
            success: '',
        }
    },
    mounted() {
        if(this.$store.state.user.userId == -1) {
            this.$router.push('/');
            return; 
        }
        const self = this; 
        this.$store.dispatch('getUserInfo', this.$store.state.user.userId)
        .then(function() {
            self.state.input.newlast_name = self.userInfo.last_name;
            self.state.input.newfirst_name = self.userInfo.first_name;
            self.state.input.newemail = self.userInfo.email;
        }, function () {
            self.logout();
        })
    },
    computed: {
        ...mapState({
            user:'user',
            userInfo: 'userInfo'
        })
    },
    methods: {
        logout: function() {
        localStorage.removeItem('user')
        this.$router.push('/');
        },
        onFileSelected(event) {
            this.selectedFile = event.target.files[0];
            let reader = new FileReader();
            reader.onload = () => {
                this.$refs.filePreview.src = reader.result;
                this.$refs.filePreview.style.display = "";
                this.$refs.photoProfil.style.display = "none";
            }
            reader.readAsDataURL(this.selectedFile);
        },
        onUpload() {
            const self = this;
            const fd = new FormData();
            fd.append('image_profil', this.selectedFile);
            this.$store.dispatch('updateImage', {
                fdImage: fd,
                userId: this.user.userId
            })
            .then(function (response) {
                self.success = response.data.message;
                self.refreshData();
            }, function (error) {
                self.error = error.response.data.error;
            })
        },
        modifyUser: function () {
            const self = this;
            this.v$.$validate();
            if (!this.v$.$error) {
                const userObjet = {
                    userId: this.user.userId,
                    user: {
                        last_name: this.state.input.newlast_name,
                        first_name: this.state.input.newfirst_name,
                        email: this.state.input.newemail
                    }
                }
                this.$store.dispatch('updateUser', userObjet
                ).then(function (response) {
                    self.success = response.data.message;
                    self.refreshData();
            }, function (error) {
                self.error = error.response.data.error;
            })
            }
        },
        deactivate() {
            const self = this;
            this.$store.dispatch('deactivateAccount', this.$store.state.user.userId 
                ).then(function () {
                self.logout();
            }, function (error) {
                self.error = error.response.data.error;
            })
        },
        goAdmin: function () {
            const self = this;
            this.v$admin.$validate();
            if (!this.v$admin.$error) {
                const info = {
                    userId: this.user.userId,
                    passwordadmin: this.state.input.passwordadmin,
                }
                this.$store.dispatch('goAdmin', info
                ).then(function (response) {
                self.success = response.data.message;
                self.refreshData();
            }, function (error) {
                self.error = error.response.data.error;
            })
            }
        },
        modifyPassword() {
            this.$router.push('/modifyPassword');
        },
        refreshData: function() {
            const self = this;
            this.$store.dispatch('getUserInfo', this.user.userId)
            .then(function () {
            }, function (error) {
                self.error = error.response.data.error;
            })
        }
}}

</script>

<style scoped>

.user-picture {
    width: 150px;
    height: 150px;
    border: 1px solid black;
    border-radius: 150px;
    margin: 0;
}

.text-profile-picture {
    font-size: 14px;
    font-style: italic;
    color: rgba(0, 0, 0, 0.5);
}

.text-admin {
    font-weight: bold;
}

.infos {
    font-weight: bold;
    text-decoration: underline;
}

.deactivate {
    font-size: 12px;
    color: red;
    border: none;
    background-color: unset;
}

.deactivate:hover {
    color: black;
}

/* RESPONSIVE MOBILE */
@media (max-width: 768px) {
    .user-picture {
        width: 100px;
        height: 100px;
    }
}

</style>