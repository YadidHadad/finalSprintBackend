<template>
    <div class="group flex column">

        <div class="main-title flex column justify-between">
            <div class="flex row align-center justify-between w-100">
                <input v-model="newGroupTitle" @input="updateGroup" @keyup.enter="($event) => $event.target.blur()" />
                <button class="btn-menu" @click="toggleMenu">
                    <span class="fa-solid elipsis-icon"></span>
                </button>
            </div>
            <!-- <span v-if="group.tasks">{{ group.tasks.length }} cards</span> -->

            <div v-if="isMenuOpen" class=" task-editor" v-click-outside="() => isMenuOpen = false">
                <section class=" title flex row justify-center">
                    <span>List actions</span>
                    <button @click="toggleMenu" class="btn-close">
                        <span class="trellicons x-icon"></span>
                    </button>
                </section>
                <div class="flex column">
                    <button @click="removeGroup" class="remove btn">
                        <span> Move list</span>
                    </button>
                    <button @click="copyGroup" class="remove btn">
                        <span> Copy list</span>
                    </button>
                    <button @click="toggleModal" class="remove btn">
                        <span> Remove list</span>
                    </button>
                </div>
            </div>
        </div>

        <Container class="task-preview-container flex column" orientation="vertical" group-name="group-tasks"
            ref="group" @drop="onDrop"
            :shouldAcceptDrop="(e, payload) => (e.groupName === 'group-tasks' && !payload.loading)"
            :get-child-payload="getChildPayload" drop-class="" :drop-class="dragClass">
            <Draggable class="task-preview" v-for="task in tasksToShow" :key="task.id">
                <task-preview :task="task" :groupId="this.group.id" :boardId="boardId" />
            </Draggable>

            <form ref="form" class="add-card-form flex" v-if="isCardOpen" @submit.prevent="addTask">
                <textarea v-model="currTask.title" type="textarea" name="add-task" rows="4"
                    placeholder="Enter a title for this card..." v-focus @keyup.enter="addTask"></textarea>
                <div class="add-list-btns flex">
                    <button class="add-list-btn">Add card</button>
                    <button type="button" @click.stop="toggleCard">
                        <span class="fa-solid x-icon"></span>
                    </button>
                </div>
            </form>
        </Container>
        <div v-if="!isCardOpen" class="add-card-container flex">
            <button class="add-card-btn" @click="toggleCard">
                <span class="fa-regular plus-icon"></span><span>Add a card</span>
            </button>
        </div>
    </div>
    <!-- <div class="confirm-modal">
            <header class="confirm-modal-header"> Are you sure ?</header>
            <div class="flex gap">
                <button class="btn-ok">Delete</button>
                <button class="btn-cancel">Go back</button>
            </div>
        </div> -->
    <confirm-modal v-if="isRemoveClicked" :msg="'Are you sure?'" @remove="removeGroup" @closeModal="toggleModal" />
</template>

<script>
import taskPreview from "../cmps/task-preview.vue"
import { utilService } from "../services/util.service.js"
import { Container, Draggable } from "vue3-smooth-dnd"
import copyTaskEdit from './copy-task-edit.vue'
import confirmModal from './confirm-modal.vue'
export default {
    name: 'group',
    emits: ["addTask", "updateGroup", "removeGroup"],
    props: {
        group: {
            type: Object,
            required: true,
        },
        boardId: {
            type: String,
        },
        filterBy: {
            type: Object,
        },
    },
    data() {
        return {
            isCardOpen: false,
            currTask: {
                id: utilService.makeId(),
                title: '',
            },
            isMenuOpen: false,
            newGroupTitle: JSON.parse(JSON.stringify(this.group.title)),
            tasksCopy: [],
            tasksToShow: [],
            dropCounter: 0,
            isRemoveClicked: false

        }
    },

    async created() {
        // console.log(this.group, '************************')
        // console.log(this.filterBy);
        this.tasksCopy = JSON.parse(JSON.stringify(this.group.tasks))
        this.tasksToShow = this.group.tasks
        this.dropDebounce = utilService.debounce(this.onDrop, 500)

        // try {
        //     this.debounceHandler = utilService.debounce(this.updateGroup, 500)

        // } catch (err) {
        //     console.log(err);
        // }
    },

    methods: {
        async onDrop(dropResult) {
            console.log('ONDROP')
            const { removedIndex, addedIndex, payload, element } = dropResult;
            if (removedIndex === null && addedIndex === null) return

            // console.log('ON DROP! - group.vue', dropResult)
            // if(addedIndex !== null) this.$store.commit({type: 'updateTasks' ,payload: { tasks: this.tasksCopy, groupId: this.group.id } })
            try {
                this.tasksCopy = JSON.parse(JSON.stringify(this.group.tasks || []))
                this.tasksCopy = this.applyDrag(this.tasksCopy, dropResult)
                // console.log('Tasks Copy', this.tasksCopy)
                await this.$store.dispatch({
                    type: 'updateTasks',
                    payload: { tasks: this.tasksCopy, groupId: this.group.id, addedIndex }
                })

                this.tasksCopy = JSON.parse(JSON.stringify(this.group.tasks || []))

            }
            catch (prevTasks) {
                console.log(prevTasks)
                this.tasksCopy = JSON.parse(JSON.stringify(prevTasks))
            }
        },
        applyDrag(arr, dragResult) {
            const { removedIndex, addedIndex, payload } = dragResult
            // console.log('PAYLOAD', payload)

            if (removedIndex === null && addedIndex === null) return arr;
            const result = [...arr];
            // console.log(result);
            let itemToAdd = payload;
            if (payload === null) return

            if (removedIndex !== null) {
                itemToAdd = result.splice(removedIndex, 1)[0];
                // console.log('ITEM-TO-ADD', itemToAdd)
            }
            if (addedIndex !== null && removedIndex !== null) {
                // console.log(itemToAdd);
                // console.log(itemToAdd.itemToMove);
                result.splice(addedIndex, 0, itemToAdd);
                // result.splice(addedIndex, 0, itemToAdd.itemToMove);
                // console.log(this.tasksCopy);
            }
            else if (addedIndex !== null) result.splice(addedIndex, 0, itemToAdd.itemToMove);
            // console.log('RESULT', result)
            return result;
        },
        getShouldAcceptDrop(index, sourceContainerOptions, payload) {
            return true;
        },

        getChildPayload(index) {
            // console.log('get child copy', index)

            this.tasksCopy = JSON.parse(JSON.stringify(this.group.tasks))

            // console.log('get child copy', this.tasksCopy);

            return {
                itemToMove: this.tasksCopy[index]
            }
        },
        updateGroup() {
            if (!this.newGroupTitle) return
            const activity = {
                id: '',
                txt: `Updated Group: ${this.group.title}`,
                byMember: {
                    _id: this.user._id,
                    fullname: this.user.fullname,
                    imgUrl: this.user.imgUrl || '',
                },
                // task: this.task
            }
            const group = JSON.parse(JSON.stringify(this.group))
            group.title = this.newGroupTitle
            // console.log(group);
            this.$emit('updateGroup', group, activity)
        },
        copyGroup() {
            this.$stor.dispatch({ type: 'addGroup', group: JSON.parse(JSON.stringify(this.group)) })
        },
        toggleCard() {
            // console.log(this.isCardOpen);
            this.isCardOpen = !this.isCardOpen;
            // this.$refs.group.containerElement['smooth-dnd-container-instance'].element.scrollTop = this.$refs.group.containerElement['smooth-dnd-container-instance'].element.scrollHeight
            // console.log(this.$refs.group.containerElement['smooth-dnd-container-instance'])
            if (this.isCardOpen) {
                setTimeout(() => {
                    console.log(this.$refs.form)
                    this.$refs.form.scrollIntoView()
                }, 100);
            }
        },
        toggleMenu() {
            this.isMenuOpen = !this.isMenuOpen;
        },
        toggleModal() {
            this.isRemoveClicked = !this.isRemoveClicked
        },
        addTask() {
            // this.$refs.group.containerElement['smooth-dnd-container-instance'].element.scrollTop = this.$refs.group.containerElement['smooth-dnd-container-instance'].element.scrollHeight + 100

            // console.log('******************************', this.user)
            if (!this.currTask.title) return
            const activity = {
                id: '',
                txt: `Added ${this.currTask.title} task to ${this.group.title}`,
                byMember: {
                    _id: this.user._id,
                    fullname: this.user.fullname,
                    imgUrl: this.user.imgUrl || '',
                },
                task: this.currTask
            }

            // this.currTask.id = utilService.makeId()
            // console.log('******************************', this.currTask)
            this.$emit('addTask', this.group.id, { ...this.currTask }, JSON.parse(JSON.stringify(activity)))
            this.currTask = {
                id: utilService.makeId(),
                title: '',
            }
        },
        removeGroup() {
            this.toggleMenu
            const activity = {
                id: '',
                txt: ` Deleted list ${this.group.title} `,
                byMember: {
                    _id: this.user._id,
                    fullname: this.user.fullname,
                    imgUrl: this.user.imgUrl || '',
                },
                task: this.currTask
            }
            this.$emit('removeGroup', this.group.id, JSON.parse(JSON.stringify(activity)))
        },
        isScroll() {
            var hasVerticalScrollbar = this.$refs.group.offsetHeight != this.$refs.group.clientHeight
            return hasVerticalScrollbar
        }

    },
    watch: {
        filterBy: {
            handler: function (filterBy, oldVal) {
                // console.log('hiiiiiiiiiiiiiiii');
                const regex = new RegExp(filterBy.title, 'i');
                this.tasksToShow = this.group.tasks.filter(task => regex.test(task.title))
                if (filterBy.isNoMembers)
                    this.tasksToShow = this.tasksToShow.filter(task => !task.memberIds?.length)
                if (filterBy.isAssignToMe)
                    this.tasksToShow = this.tasksToShow.filter(task => task.memberIds?.includes(this.user._id))
                if (filterBy.membersIds.length) {
                    this.tasksToShow = this.tasksToShow.filter(task => {
                        if (!task.memberIds?.length) return false
                        return task.memberIds.some(memberId => filterBy.membersIds.includes(memberId))
                        // task.memberIds?.includes(this.user._id)
                    })
                }
                if (filterBy.isNoLabels) {
                    this.tasksToShow = this.tasksToShow.filter(task => !task.labelIds?.length)
                }
                if (filterBy.labelIds.length) {
                    this.tasksToShow = this.tasksToShow.filter(task => {
                        if (!task.labelIds?.length) return false
                        return task.labelIds.some(labelId => filterBy.labelIds.includes(labelId))
                    })
                }
                // console.log(this.tasksToShow);
            },
            deep: true
        },
        group: {
            handler: function (val, oldVal) {
                this.tasksToShow = this.group.tasks
            },
            deep: true
        }
    },

    computed: {
        user() {
            return this.$store.getters.loggedinUser

        },
        dragClass() {
            return 'on-drag'
        }
    },
    components: { taskPreview, Container, Draggable, copyTaskEdit, confirmModal },
};
</script>
