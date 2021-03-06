<template>

  <div>
    <h1>{{ $tr('createNewExam') }}</h1>
    <template v-if="!inSearchMode">
      <KTextbox
        ref="title"
        :label="$tr('title')"
        :autofocus="true"
        :invalid="titleIsInvalid"
        :invalidText="titleIsInvalidText"
        :maxlength="100"
        @blur="titleBlurred = true"
        v-model.trim="examTitle"
      />
      <KTextbox
        ref="numQuest"
        type="number"
        :label="$tr('numQuestions')"
        :invalid="numQuestIsInvalid"
        :invalidText="numQuestIsInvalidText"
        @blur="numQuestBlurred = true"
        v-model.trim.number="examNumberOfQuestions"
      />

      <UiAlert
        v-if="selectionIsInvalid"
        type="error"
        :dismissible="false"
      >
        {{ selectionIsInvalidText }}
      </UiAlert>

      <div class="buttons-container">
        <KButton :text="$tr('preview')" @click="preview" />
        <KButton
          :text="$tr('saveButtonlabel')"
          :primary="true"
          @click="finish"
          :disabled="submitting"
        />
      </div>
    </template>

    <KGrid>
      <KGridItem
        sizes="100, 50, 50"
        percentage
      >
        <h2>{{ $tr('chooseExercises') }}</h2>

      </KGridItem>

      <KGridItem
        sizes="100, 50, 50"
        percentage
        alignments="left, right, right"
      >
        <p>{{ $tr('selected', { count: selectedExercises.length }) }}</p>
      </KGridItem>
    </KGrid>

    <LessonsSearchBox
      class="search-box"
      @searchterm="handleSearchTerm"
    />

    <template v-if="inSearchMode">
      <KButton
        :text="$tr('exitSearchButtonLabel')"
        appearance="raised-button"
        @click="handleExitSearch"
      />

      <LessonsSearchFilters
        :searchTerm="searchTerm"
        :searchResults="searchResults"
        v-model="filters"
      />
    </template>

    <ResourceSelectionBreadcrumbs
      v-else
      :ancestors="ancestors"
      :channelsLink="channelsLink"
      :topicsLink="topicsLink"
    />

    <ContentCardList
      :contentList="filteredContentList"
      :showSelectAll="selectAllIsVisible"
      :viewMoreButtonState="viewMoreButtonState"
      :selectAllChecked="selectAllChecked"
      :selectAllIndeterminate="selectAllIndeterminate"
      :contentIsChecked="contentIsSelected"
      :contentIsIndeterminate="contentIsIndeterminate"
      :contentHasCheckbox="contentHasCheckbox"
      :contentCardMessage="selectionMetadata"
      :contentCardLink="contentLink"
      @changeselectall="toggleTopicInWorkingResources"
      @change_content_card="toggleSelected"
      @moreresults="handleMoreResults"
    />

    <PreviewNewExamModal
      v-if="showPreviewNewExamModal"
      :examQuestionSources="questionSources"
      :examSeed="examSeed"
      :examNumQuestions="examNumberOfQuestions"
      :exerciseContentNodes="selectedExercises"
      @randomize="randomize"
      @close="setExamsModal(null)"
    />

  </div>

</template>


<script>

  import { mapState, mapActions, mapMutations, mapGetters } from 'vuex';

  import { ContentNodeKinds } from 'kolibri.coreVue.vuex.constants';
  import responsiveWindow from 'kolibri.coreVue.mixins.responsiveWindow';
  import KButton from 'kolibri.coreVue.components.KButton';
  import KTextbox from 'kolibri.coreVue.components.KTextbox';
  import UiAlert from 'kolibri.coreVue.components.UiAlert';
  import KGrid from 'kolibri.coreVue.components.KGrid';
  import KGridItem from 'kolibri.coreVue.components.KGridItem';

  import uniqBy from 'lodash/uniqBy';
  import shuffle from 'lodash/shuffle';
  import orderBy from 'lodash/orderBy';
  import random from 'lodash/random';
  import flatMap from 'lodash/flatMap';
  import pickBy from 'lodash/pickBy';

  import { PageNames } from '../../../constants/';
  import { Modals as ExamModals } from '../../../constants/examConstants';
  import LessonsSearchBox from '../../lessons/LessonResourceSelectionPage/SearchTools/LessonsSearchBox';
  import LessonsSearchFilters from '../../lessons/LessonResourceSelectionPage/SearchTools/LessonsSearchFilters';
  import ResourceSelectionBreadcrumbs from '../../lessons/LessonResourceSelectionPage/SearchTools/ResourceSelectionBreadcrumbs';
  import ContentCardList from '../../lessons/LessonResourceSelectionPage/ContentCardList';
  import PreviewNewExamModal from './PreviewNewExamModal';

  export default {
    // TODO: Rename this to 'ExamCreationPage'
    name: 'CreateExamPage',
    metaInfo() {
      return {
        title: this.$tr('documentTitle'),
      };
    },
    components: {
      KButton,
      KTextbox,
      PreviewNewExamModal,
      UiAlert,
      LessonsSearchBox,
      LessonsSearchFilters,
      ResourceSelectionBreadcrumbs,
      ContentCardList,
      KGrid,
      KGridItem,
    },
    mixins: [responsiveWindow],
    $trs: {
      createNewExam: 'Create new exam',
      chooseExercises: 'Select topics or exercises',
      title: 'Title',
      numQuestions: 'Number of questions',
      examRequiresTitle: 'This field is required',
      numQuestionsBetween: 'Enter a number between 1 and 50',
      numQuestionsExceed:
        'The max number of questions based on the exercises you selected is {maxQuestionsFromSelection}. Select more exercises to reach {inputNumQuestions} questions, or lower the number of questions to {maxQuestionsFromSelection}.',
      numQuestionsNotMet:
        'Add more exercises to reach 40 questions. Alternately, lower the number of exam questions.',
      noneSelected: 'No exercises are selected',
      preview: 'Preview',
      saveButtonlabel: 'Save',
      // TODO: Interpolate strings correctly
      added: 'Added',
      removed: 'Removed',
      selected: '{count, number, integer} total selected',
      documentTitle: 'Create new exam',
      exitSearchButtonLabel: 'Exit search',
      // TODO: Handle singular/plural
      selectionInformation:
        '{count, number, integer} of {total, number, integer} resources selected',
    },
    data() {
      return {
        examTitle: '',
        examNumberOfQuestions: '',
        examSeed: null,
        titleBlurred: false,
        numQuestBlurred: false,
        selectionMade: false,
        previewOrSubmissionAttempt: false,
        submitting: false,
        moreResultsState: null,
        // null corresponds to 'All' filter value
        filters: {
          channel: this.$route.query.channel || null,
          kind: this.$route.query.kind || null,
          role: this.$route.query.role || null,
        },
      };
    },
    computed: {
      ...mapState(['classId', 'pageName']),
      ...mapGetters({
        channels: 'getChannels',
      }),
      ...mapGetters('examCreation', ['numRemainingSearchResults']),
      ...mapState('examCreation', [
        'title',
        'numberOfQuestions',
        'seed',
        'contentList',
        'selectedExercises',
        'availableQuestions',
        'searchResults',
        'ancestors',
        'examsModalSet',
      ]),
      filteredContentList() {
        const { role } = this.filters || {};
        if (!this.inSearchMode) {
          return this.contentList;
        }
        return this.searchResults.results.filter(contentNode => {
          let passesFilters = true;
          if (role === 'nonCoach') {
            passesFilters = passesFilters && contentNode.num_coach_contents === 0;
          }
          if (role === 'coach') {
            passesFilters = passesFilters && contentNode.num_coach_contents > 0;
          }
          return passesFilters;
        });
      },
      allExercises() {
        const topics = this.contentList.filter(({ kind }) => kind === ContentNodeKinds.TOPIC);
        const exercises = this.contentList.filter(({ kind }) => kind === ContentNodeKinds.EXERCISE);
        const topicExercises = flatMap(topics, ({ exercises }) => exercises);
        return [...exercises, ...topicExercises];
      },
      addableExercises() {
        return this.allExercises.filter(
          exercise =>
            this.selectedExercises.findIndex(
              selectedExercise => selectedExercise.id === exercise.id
            ) === -1
        );
      },
      selectAllChecked() {
        return this.addableExercises.length === 0;
      },
      selectAllIndeterminate() {
        if (this.selectAllChecked) {
          return false;
        }
        return this.addableExercises.length !== this.allExercises.length;
      },
      questionSources() {
        const questionSources = [];
        for (let i = 0; i < this.examNumberOfQuestions; i++) {
          const questionSourcesIndex = i % this.uniqueSelectedExercises.length;
          if (questionSources[questionSourcesIndex]) {
            questionSources[questionSourcesIndex].number_of_questions += 1;
          } else {
            questionSources.push({
              exercise_id: this.uniqueSelectedExercises[i].id,
              number_of_questions: 1,
              title: this.uniqueSelectedExercises[i].title,
            });
          }
        }
        return orderBy(questionSources, [exercise => exercise.title.toLowerCase()]);
      },
      inSearchMode() {
        return this.pageName === PageNames.EXAM_CREATION_SEARCH;
      },
      searchTerm() {
        return this.$route.params.searchTerm;
      },
      selectAllIsVisible() {
        return !this.inSearchMode && this.pageName !== PageNames.EXAM_CREATION_ROOT;
      },
      viewMoreButtonState() {
        if (!this.inSearchMode) {
          return 'no_more_results';
        }
        if (this.moreResultsState === 'waiting' || this.moreResultsState === 'error') {
          return this.moreResultsState;
        }
        if (!this.numRemainingSearchResults) {
          return 'no_more_results';
        }
        return 'visible';
      },

      titleIsInvalidText() {
        if (this.titleBlurred || this.previewOrSubmissionAttempt) {
          if (this.examTitle === '') {
            return this.$tr('examRequiresTitle');
          }
        }
        return '';
      },
      titleIsInvalid() {
        return Boolean(this.titleIsInvalidText);
      },
      numQuestExceedsSelection() {
        return this.examNumberOfQuestions > this.availableQuestions;
      },
      exercisesAreSelected() {
        return this.selectedExercises.length > 0;
      },
      numQuestIsInvalidText() {
        if (this.numQuestBlurred || this.previewOrSubmissionAttempt) {
          if (this.examNumberOfQuestions === '') {
            return this.$tr('numQuestionsBetween');
          }
          if (this.examNumberOfQuestions < 1 || this.examNumberOfQuestions > 50) {
            return this.$tr('numQuestionsBetween');
          }
          if (!Number.isInteger(this.examNumberOfQuestions)) {
            return this.$tr('numQuestionsBetween');
          }
          if (this.exercisesAreSelected && this.numQuestExceedsSelection) {
            return this.$tr('numQuestionsExceed', {
              inputNumQuestions: this.examNumberOfQuestions,
              maxQuestionsFromSelection: this.availableQuestions,
            });
          }
        }
        return '';
      },
      numQuestIsInvalid() {
        return Boolean(this.numQuestIsInvalidText);
      },
      selectionIsInvalidText() {
        if (this.selectionMade || this.previewOrSubmissionAttempt) {
          if (!this.exercisesAreSelected) {
            return this.$tr('noneSelected');
          }
        }
        return '';
      },
      selectionIsInvalid() {
        return Boolean(this.selectionIsInvalidText);
      },
      formIsInvalidText() {
        if (this.titleIsInvalid) {
          return this.titleIsInvalidText;
        }
        if (this.numQuestIsInvalid) {
          return this.numQuestIsInvalidText;
        }
        if (this.selectionIsInvalid) {
          return this.selectionIsInvalidText;
        }
        return '';
      },
      formIsInvalid() {
        return Boolean(this.formIsInvalidText);
      },
      uniqueSelectedExercises() {
        return uniqBy(this.selectedExercises, 'content_id');
      },
      showPreviewNewExamModal() {
        return this.examsModalSet === ExamModals.PREVIEW_NEW_EXAM;
      },
      channelsLink() {
        return {
          name: PageNames.EXAM_CREATION_ROOT,
          params: {
            classId: this.classId,
          },
        };
      },
    },
    watch: {
      filters(newVal) {
        this.$router.push({
          query: pickBy(newVal),
        });
      },
    },
    created() {
      this.examTitle = this.title;
      this.examNumberOfQuestions = this.numberOfQuestions;
      this.examSeed = this.seed || this.generateRandomSeed();
      this.$watch('examTitle', () => this.setTitle(this.examTitle));
      this.$watch('examNumberOfQuestions', () =>
        this.setNumberOfQuestions(this.examNumberOfQuestions)
      );
      this.$watch('examSeed', () => this.setSeed(this.examSeed));
    },
    methods: {
      ...mapActions(['createSnackbar']),
      ...mapActions('examCreation', [
        'addToSelectedExercises',
        'removeFromSelectedExercises',
        'setSelectedExercises',
        'fetchAdditionalSearchResults',
        'createExamAndRoute',
      ]),
      ...mapMutations('examCreation', {
        setExamsModal: 'SET_EXAMS_MODAL',
        setTitle: 'SET_TITLE',
        setNumberOfQuestions: 'SET_NUMBER_OF_QUESTIONS',
        setSeed: 'SET_SEED',
      }),
      contentLink(content) {
        if (content.kind === ContentNodeKinds.TOPIC || content.kind === ContentNodeKinds.CHANNEL) {
          return {
            name: PageNames.EXAM_CREATION_TOPIC,
            params: {
              classId: this.classId,
              topicId: content.id,
            },
          };
        }
        return {
          name: PageNames.EXAM_CREATION_PREVIEW,
          params: {
            classId: this.classId,
            contentId: content.id,
          },
        };
      },
      contentHasCheckbox() {
        return this.pageName === PageNames.EXAM_CREATION_ROOT;
      },
      contentIsSelected(content) {
        if (content.kind === ContentNodeKinds.TOPIC) {
          return content.exercises.every(
            exercise =>
              this.selectedExercises.findIndex(
                selectedExercise => selectedExercise.id === exercise.id
              ) !== -1
          );
        } else {
          return (
            this.selectedExercises.findIndex(
              selectedExercise => selectedExercise.id === content.id
            ) !== -1
          );
        }
      },
      contentIsIndeterminate(content) {
        if (content.kind === ContentNodeKinds.TOPIC) {
          const everyExerciseSelected = content.exercises.every(
            exercise =>
              this.selectedExercises.findIndex(
                selectedExercise => selectedExercise.id === exercise.id
              ) !== -1
          );
          if (everyExerciseSelected) {
            return false;
          }
          const someExerciseSelected = content.exercises.some(
            exercise =>
              this.selectedExercises.findIndex(
                selectedExercise => selectedExercise.id === exercise.id
              ) !== -1
          );
          return someExerciseSelected;
        }
        return false;
      },
      selectionMetadata(content) {
        if (content.kind === ContentNodeKinds.TOPIC) {
          const count = content.exercises.filter(
            exercise =>
              this.selectedExercises.findIndex(
                selectedExercise => selectedExercise.id === exercise.id
              ) !== -1
          ).length;
          if (count === 0) {
            return '';
          }
          const total = content.exercises.length;
          return this.$tr('selectionInformation', { count, total });
        }
        return '';
      },
      toggleTopicInWorkingResources(isChecked) {
        const topicTitle = this.ancestors[this.ancestors.length - 1].title;
        if (isChecked) {
          this.addToSelectedExercises(this.addableExercises);
          this.createSnackbar({ text: `${this.$tr('added')} ${topicTitle}`, autoDismiss: true });
        } else {
          this.removeFromSelectedExercises(this.allExercises);
          this.createSnackbar({ text: `${this.$tr('removed')} ${topicTitle}`, autoDismiss: true });
        }
      },
      toggleSelected({ checked, contentId }) {
        let exercises;
        const contentNode = this.contentList.find(item => item.id === contentId);
        const isTopic = contentNode.kind === ContentNodeKinds.TOPIC;
        if (checked && isTopic) {
          exercises = contentNode.exercises;
          this.addToSelectedExercises(exercises);
          this.createSnackbar({
            text: `${this.$tr('added')} ${contentNode.title}`,
            autoDismiss: true,
          });
        } else if (checked && !isTopic) {
          exercises = [contentNode];
          this.addToSelectedExercises(exercises);
          this.createSnackbar({
            text: `${this.$tr('added')} ${contentNode.title}`,
            autoDismiss: true,
          });
        } else if (!checked && isTopic) {
          exercises = contentNode.exercises;
          this.removeFromSelectedExercises(exercises);
          this.createSnackbar({
            text: `${this.$tr('removed')} ${contentNode.title}`,
            autoDismiss: true,
          });
        } else if (!checked && !isTopic) {
          exercises = [contentNode];
          this.removeFromSelectedExercises(exercises);
          this.createSnackbar({
            text: `${this.$tr('removed')} ${contentNode.title}`,
            autoDismiss: true,
          });
        }
      },
      handleMoreResults() {
        this.moreResultsState = 'waiting';
        this.fetchAdditionalSearchResults({
          searchTerm: this.searchTerm,
          kind: this.filters.kind,
          channelId: this.filters.channel,
          currentResults: this.searchResults.results,
        })
          .then(() => {
            this.moreResultsState = null;
          })
          .catch(() => {
            this.moreResultsState = 'error';
          });
      },
      handleExitSearch() {
        const lastId = this.$route.query.last_id;
        if (lastId) {
          this.$router.push({
            name: PageNames.EXAM_CREATION_TOPIC,
            params: {
              classId: this.classId,
              topicId: lastId,
            },
          });
        } else {
          this.$router.push({
            name: PageNames.EXAM_CREATION_ROOT,
            params: {
              classId: this.classId,
            },
          });
        }
      },
      preview() {
        this.previewOrSubmissionAttempt = true;
        if (this.formIsInvalid) {
          this.focusOnInvalidField();
        } else {
          this.setExamsModal(ExamModals.PREVIEW_NEW_EXAM);
        }
      },
      finish() {
        this.previewOrSubmissionAttempt = true;
        if (this.formIsInvalid) {
          this.focusOnInvalidField();
        } else {
          this.submitting = true;
          const dummyChannelId = this.channels[0].id;
          const exam = {
            collection: this.classId,
            channel_id: dummyChannelId,
            title: this.examTitle,
            question_count: this.examNumberOfQuestions,
            question_sources: this.questionSources,
            seed: this.examSeed,
            assignments: [{ collection: this.classId }],
          };
          this.createExamAndRoute(exam);
        }
      },
      focusOnInvalidField() {
        if (this.titleIsInvalid) {
          this.$refs.title.focus();
        } else if (this.numQuestIsInvalid) {
          this.$refs.numQuest.focus();
        }
      },
      generateRandomSeed() {
        return random(1000);
      },
      randomize() {
        this.examSeed = this.generateRandomSeed();
        this.setSelectedExercises(shuffle(this.selectedExercises));
      },
      handleSearchTerm(searchTerm) {
        const lastId = this.$route.query.last_id || this.$route.params.topicId;
        this.$router.push({
          name: PageNames.EXAM_CREATION_SEARCH,
          params: {
            searchTerm,
          },
          query: {
            last_id: lastId,
          },
        });
      },
      topicsLink(topicId) {
        return {
          name: PageNames.EXAM_CREATION_TOPIC,
          params: {
            classId: this.classId,
            topicId,
          },
        };
      },
    },
  };

</script>


<style lang="scss" scoped>

  @import '~kolibri.styles.definitions';

  .search-box {
    display: inline-block;
    vertical-align: middle;
  }

  .buttons-container {
    text-align: right;
    button {
      margin: 0 0 0 16px;
    }
  }

</style>
