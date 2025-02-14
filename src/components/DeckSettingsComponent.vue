<template>

    <div class="container" :class="breakpoints">
        <div @click="saveDeck" :class="{ 'active': !isSaved && !isSaving, 'process': isSaving }"
            class="save-btn pointer">
            <BIconFileEarmarkFill class="save-btn-icon" />
        </div>

        <div class="deck-name-container">
            <div class="deck-name">
                <base-switchable-input v-model="setupedDeckName" @update:modelValue="isNameSaved = false"
                    :placeholder="'Пусто'" />
            </div>

        </div>
        <div class="form-container">
            <DeckSettingsForm :step="step" :handleAddToFront="handleAddToFront" :handleAddToBack="handleAddToBack"
                :handleDeleteFromFront="handleDeleteFromFront" :handleDeleteFromBack="handleDeleteFromBack"
                v-model="structure" class="form" />
            <DeckUpdatePreview :front="preview.front" :back="preview.back" class="preview" />
        </div>
        <div class="table-container">
            <DeckSettingsTable v-model="tableDataForSave" :handleDeleteRow="handleDeleteRow" :headers="headers"
                :data="data.data" />
        </div>
    </div>

</template>
  
<script>
import { BIconFileEarmarkFill } from "bootstrap-icons-vue";
import DeckSettingsForm from "@/components/DeckSettingsForm";
import DeckUpdatePreview from "@/components/DeckUpdatePreview";
import DeckSettingsTable from "@/components/DeckSettingsTable";
import BaseSwitchableInput from "@/components/BaseSwitchableInput"
import { useDeckSettingsForm, useTable, useDeck } from "@/hooks";
import { useRoute } from "vue-router";
import { useToast } from "vue-toastification";
import { Api } from "@/api"
import { breakpointsMixin } from "@/mixins";

export default {
    name: "DeckSettingsPage",
    components: {
        DeckSettingsForm,
        DeckUpdatePreview,
        DeckSettingsTable,
        BaseSwitchableInput,
        BIconFileEarmarkFill
    },

    mixins: [breakpointsMixin],

    methods: {
        checkSameAndEmptyValues(array) {
            const fieldNames = [];

            for (let field of array) {
                if (field.name && !fieldNames.includes(field.name.toLowerCase())) {
                    fieldNames.push(field.name.toLowerCase());
                } else return false;
            }

            return true
        },

        async saveDeck() {
            this.isSaving = true;


            const dbStructure = this.getRawStructure(this.structure);
            if (!this.setupedDeckName) {
                this.toast.error(`Нет названия колоды`, {
                    timeout: 2000,
                });
                this.isSaving = false;
                return;
            }

            if (!this.checkSameAndEmptyValues(dbStructure)) {
                this.toast.error(`Пустые названия в структуре колоды `, {
                    timeout: 2000,
                });
                this.isSaving = false;
                return;
            }


            //Create deck if not
            if (!this.deckSlug) {
                const [error, data] = await Api().createDeck(this.setupedDeckName);
                if (error) {
                    this.toast.error(`Ошибка случилась : ${error}`, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                    return;
                }
                this.deckSlug = data;
                this.isNameSaved = true
            }
            //Save name
            else if (!this.isNameSaved) {
                const [error, data] = await Api().updateDeckName(this.setupedDeckName, this.deckSlug);
                if (error) {
                    this.toast.error(`Ошибка случилась : ${error}`, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                }
                this.deckSlug = data;
                this.isNameSaved = true
            }
            this.$router.replace(`/settings/${this.deckSlug}`)

            //Save fields
            if (!this.isStructureSaved) {
                const [error] = await Api().updateDeckFields(dbStructure, this.deckSlug);
                if (error) {
                    this.toast.error(`Ошибка случилась : ${error}`, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                }
                this.isStructureSaved = true;
            }
            //Save values
            if (this.tableDataForSave.length) {
                let notGood = false;
                //empty values for every field in one side is not good
                this.tableDataForSave.forEach(item => {
                    const card = this.data.data.find(card => item.cardId === card.id);
                    if (card) {
                        const values = Object.values(card);
                        values.shift();
                        const frontSide = values.filter(value => value.field.side === "front")
                        const backSide = values.filter(value => value.field.side === "back")

                        if (!frontSide.find(value => value.value) || !backSide.find(value => value.value)) {
                            notGood = true;
                        }

                    }
                })

                if (notGood) {
                    this.toast.error(`Одна из карт содержит пустые поля с одной из сторон `, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                    return;
                }
                const [error] = await Api().updateDeckCards(this.tableDataForSave, this.deckSlug);
                if (error) {
                    this.toast.error(`Ошибка случилась : ${error}`, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                }
                this.tableDataForSave = [];
            }
            //Remove cards
            if (this.tableCardsForRemove.length) {
                const [error] = await Api().removeCards(this.tableCardsForRemove);
                if (error) {
                    this.toast.error(`Ошибка случилась : ${error}`, {
                        timeout: 2000,
                    });
                    this.isSaving = false;
                }
                this.tableCardsForRemove = [];

            }

            this.isSaving = false;


        }
    },


    async setup() {
        const route = useRoute();
        const deckSlug = route.params.deckSlug;
        const toast = useToast();


        const { getStructuredDeck, getRawStructure, getDefaultDeck } = useDeck();


        const getExistedOrDefaultDeck = async () => {
            if (deckSlug) {
                const data = await getStructuredDeck(deckSlug, (item) => ({
                    ...item,
                    type: {
                        accessor: item.type,
                        name: item.type === "main" ? "Больше" : "Меньше",
                    }
                }));
                return data;

            } else {
                return getDefaultDeck((item) => ({
                    ...item,
                    type: {
                        accessor: item.type,
                        name: item.type === "main" ? "Больше" : "Меньше",
                    }
                }));
            }
        }
        const deckStructureToTableStructure = (deckStr) => {
            return deckStr.front.concat(deckStr.back).map((item) => ({
                ...item,
                accessor: item.name.toLowerCase() + "_" + item.id,
            }));
        }

        const { name, dbStructure, cards } = await getExistedOrDefaultDeck();

        const {
            setupedDeckName,
            structure,
            handleAddToFront,
            handleAddToBack,
            handleDeleteFromFront,
            handleDeleteFromBack,
            step,
            structureWatcher,
            isStructureSaved,
        } = await useDeckSettingsForm(name, dbStructure);

        const {
            updateStructure,
            data,
            headers,
            handleDeleteRow,
            tableDataForSave,
            tableCardsForRemove
        } = useTable(deckStructureToTableStructure(structure), cards);

        structureWatcher((newValue) => {
            const newStructure = deckStructureToTableStructure(newValue);
            updateStructure(newStructure);
        })

        return {
            handleAddToFront,
            handleAddToBack,
            handleDeleteFromFront,
            handleDeleteFromBack,
            structure,
            data,
            headers,
            step,
            handleDeleteRow,
            setupedDeckName,
            tableDataForSave,
            isStructureSaved,
            getRawStructure,
            tableCardsForRemove,
            toast
        };
    },



    data() {
        return {
            deckSlug: this.$route.params.deckSlug,
            isNameSaved: true,
            isSaving: false,
        }
    },

    computed: {
        isSaved() {
            return this.isNameSaved
                && this.isStructureSaved
                && !this.tableDataForSave.length
                && !this.tableCardsForRemove.length;
        },
        preview() {
            return {
                front: this.structure.front.map((item) => ({
                    ...item,
                    type: item.type.accessor,
                    value: item.name
                })),
                back: this.structure.back.map((item) => ({
                    ...item,
                    type: item.type.accessor,
                    value: item.name
                })),
            }
        }
    },
};
</script>
  
<style lang="scss" scoped>
@keyframes save-btn-appear {
    from {
        transform: scale(0.5);
    }

    33% {
        visibility: visible;
        opacity: 1;
        transform: scale(1.25)
    }

    66% {
        visibility: visible;
        opacity: 1;
        transform: scale(0.8)
    }

    to {
        transform: scale(1.0)
    }
}

@keyframes save-btn-active {


    6% {
        transform: scale(1.25) rotate(-30deg)
    }

    13% {
        transform: scale(1.25) rotate(30deg)
    }

    19.999% {
        transform: scale(1.0) rotate(0deg)
    }

    20% {
        transform: scale(1.0) rotate(0deg)
    }
}

@keyframes save-btn-process {


    33% {
        transform: scale(1.25) rotate(360deg)
    }

    66% {
        transform: scale(1.25) rotate(720deg)
    }

    99.999% {
        transform: scale(1.25) rotate(1080deg)
    }

    100% {
        transform: scale(1.25) rotate(0deg)
    }
}

.loading {
    font-size: 5em;
    font-weight: bold;
}

.save-btn {
    position: fixed;
    font-size: 1.5em;
    bottom: 50px;
    right: 50px;
    z-index: 1000;
    background: $primary-text;
    border-radius: 50%;
    padding: 15px;
    display: flex;
    text-align: center;
    justify-content: center;
    align-items: center;
    visibility: hidden;
    opacity: 0;

    .save-btn-icon {
        fill: white;
    }

    &.active {
        animation: save-btn-appear 0.5s linear, save-btn-active ease-in-out 5s infinite 2s;
        visibility: visible;
        opacity: 1;
    }

    &.process {
        visibility: visible;
        opacity: 1;
        animation: save-btn-process 1.5s ease-in-out infinite;
        transition: transform 0.4s;
        transform: scale(1.25)
    }
}

.container {
    .deck-name-container {
        width: 100%;
        align-self: start;
        font-size: 3em;
        font-weight: bold;


        .deck-name {
            max-width: 100%;
            padding-left: 30px;
            padding-right: 30px;
            word-break: normal;
            overflow: hidden;
            white-space: nowrap;
            text-overflow: ellipsis;
        }
    }

    display: flex;
    flex-direction: column;
    align-items: center;
    height: calc(100vh - 80px);
    padding: 40px 0;

    .form-container {
        margin-top: 30px;
        width: 100%;
        display: flex;
        justify-content: space-between;
        align-items: center;
        justify-content: space-between;

        .form {
            width: 45vw;
        }

        .preview {
            width: 35vw;
            margin-right: 30px;
        }
    }

    .table-container {
        width: 90%;
        margin-top: 100px;
    }

    &.lg {
        .form-container {
            .form {
                width: 55vw;
            }

            .preview {

                margin-right: 0;
            }
        }
    }

    &.md {
        .form-container {
            flex-direction: column;

            .form {
                width: 90vw;
                min-height: 100%;
                height: auto;
            }

            .preview {
                display: none;
            }
        }

        .save-btn {
            bottom: 20px;
            right: 20px;
        }
    }
}
</style>