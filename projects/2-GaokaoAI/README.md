# 【教程】基于智谱清言，创建一个写作今年高考作文 + 批改的 AI 应用

> 笔者目前正在挑战基于国内的大模型创建 10个AI应用，相关的连载教程 + Demo体验地址会在[Github仓库](https://github.com/KapiAI/KapiAI-Apps)持续更新。
>
> 这是基于国内大模型创建 AI 应用的第 2 期。AI 应用的开发过程会同步到 B 站： [@卡皮 AI](https://space.bilibili.com/39930228)，欢迎大家关注。


## 前言

![项目示例](https://static-main.aiyeshi.cn/images/gaokao-202406/271shots_so.jpg)

本期开发的应用是一个用智谱清言的 API 接口，开发的一个让 AI 来完成今年的高考作文的写作，并实现 AI 批改评分的一个 AI 应用。

整个应用的 UI 部分，有 3 个部分：

- 头部：试卷的切换功能，可以切换选择不同省份的试卷详情。主体部分采用左右分布的一个布局。
- 左侧部分：对应各个省份的作文题目，以及由 AI 根据该作为题目写作的作文内容。
- 右侧部分：展示了 AI 基于作文题目和作文内容，给出的具体评分，包括分数、作文等级和打分的依据。

## 大模型 Prompt 开发

首先我们直接进入到大模型的后端开发，这个 AI 应用的核心的接口就 2 个：让 AI 写作、让 AI 判分。

智谱清言的 API 调用是兼容了 OpenAI 的 npm 库，所以在开发后端应用之前，记得提前在智谱清言的后台获取到你的 API_KEY 和对应的 BASE_URL，然后实例化 OpenAI。

```javascript
import OpenAI from "openai";

const openai = new OpenAI({
  apiKey: process.env["GLM_API_KEY"],
  baseURL: process.env["GLM_BASE_URL"],
});
```

### AI 写作

![写作功能](https://static-main.aiyeshi.cn/images/gaokao-202406/gaokao-ai-write_button.png)

首先我们进入到 AI 写作的开发，用户点击【AI 写作】按钮后，会调用`/api/gaokao/write`接口，接口接收两个参数：

- `city`: 城市名称
- `question`：作文题目信息

接口的 Controller 代码如下：

```javascript
@Post('/write')
async getWrite(@Body() body) {
  const { city, question } = body;

  const data = await this.gaokaoService.getAIEssay({ city, question });

  return {
    success: !!data,
    data,
    info: !!data ? null : '生成失败',
  };
}
```

我们的核心调用智谱清言的核心 Prompt 的逻辑，封装在对应的 Service 中。

```javascript
async getAIEssay({ question, city }) {
  // 构建 Prompt
  const prompt = `你现在是一名正在参加${city}高考的考生，你正在写作语文高考作文。要求：
  1. 文章要符合题目要求，要有明确的中心思想，要有引子、承接、转折、结尾等要素。
  2. 文章需要满足高考作文的字数要求，字数不够会扣分。
  3. 文章需要有较好的语言表达，不要出现语法错误。
  4. 文章需要有较好的文采，不要出现重复、啰嗦等问题。
  5. 文章需要有较好的逻辑性，不要出现逻辑混乱等问题。
  6. 文章需要有较好的内容，不要出现内容空洞等问题。
  7. 文章需要有较好的结构，不要出现结构混乱等问题。
  8. 文章需要有较好的主题，不要偏离主题等问题。
  9. 文章需要有较好的观点，不要出现观点不明确等问题。
  10. 文章需要有一定的思想、并且有感情真实
  10. 请直接开始从题目开始写，题目使用<title><title/>进行包裹，无需其他的回答

  请根据以下题目撰写一篇高考作文：
  题目：${question}`;

  try {
    const chatCompletion: OpenAI.Chat.ChatCompletion =
      await openai.chat.completions.create({
        model: 'glm-4',
        messages: [
          { role: 'system', content: '你是一个高考考生。' },
          { role: 'user', content: prompt },
        ],
        temperature: 0.7,
      });

    // 返回结果
    return chatCompletion?.choices[0]?.message?.content || '';
  } catch (error) {
    return error.message;
  }
}
```

为了让生成的文章的内容更加出彩，我们这边调用的是智谱清言 GLM-4 的模型。在[智谱清言官网](https://open.bigmodel.cn/dev/howuse/glm-4)中，GLM-4 的介绍如下：

> 新一代基座大模型 GLM-4，整体性能相比 GLM3 全面提升 60%，逼近 GPT-4；支持更长上下文；更强的多模态；支持更快推理速度，更多并发，大大降低推理成本；同时 GLM-4 增强了智能体能力。

在 Prompt 中，我们在`system`中让它假设自己是一个高考生，然后在`user`中，让它告知了大模型的高考作文题目。以及写文章需要注意的要点和注意事项。

在这个 Prompt 中需要注意的是，在第 10 条中，我们让大模型直接给出回答，无需其他的描述，这样确保了大模型给出的回答只有标题和作文的内容。我们还让大模型将作文的标题使用`<title><title/>`进行报告包裹，方便之后对标题如果有特殊样式的前端处理，方便实现正则的一个替换工作。

以*新课标 I 卷*为测试，如果调用成功，大模型返回的数据结果如下：

```
<title>智慧的探求：问题与答案的辩证法</title>

在这个互联网和人工智能迅猛发展的时代，知识的获取变得前所未有地便捷。一个问题只需轻轻敲击键盘，就能在瞬间得到答案。然而，这并不意味着我们的问题会越来越少，相反，我认为，问题和答案之间存在着一种辩证的关系，它们在探求智慧的道路上相辅相成。

科技的进步确实在很大程度上解决了我们的疑问，但同时也引发了更多的问题。互联网像一个无尽的宝库，让我们在探索中不断发现新的知识领域。当我们对一个答案感到满意时，新的问题往往接踵而至。这种循环往复的过程促使我们不断进步，不断挑战自己的认知边界。

在我看来，问题并非负担，而是推动我们前进的动力。每一个问题都代表着一个新的思考方向，引领我们走向更深层次的认知。正如古人所说：“学起于思，思源于疑。”疑问是我们探求真理的起点，也是我们成长过程中不可或缺的元素。

然而，我们不能仅仅满足于获取答案，更要学会独立思考。在这个信息爆炸的时代，过多的答案往往会让人陷入迷茫。我们需要具备辨别真伪、筛选信息的能力，从而形成自己的观点。这就要求我们在探求答案的过程中，不仅要关注结果，更要关注方法，培养自己的思维能力。

在这个背景下，我认为我们的问题不会越来越少，而是会越来越多。因为随着认知水平的提高，我们会站在更高的角度去审视这个世界，发现更多未知的问题。同时，人工智能和互联网的发展也在不断拓宽我们的视野，让我们有机会接触到更多领域的知识，从而产生更多的问题。

那么，如何在这个充满问题的世界找到自己的方向呢？我认为，首先要保持好奇心。好奇心是我们探索未知的驱动力，它能激发我们的求知欲，让我们在面对问题时保持积极的态度。其次，要勇于质疑。不盲从权威，不满足于表面的答案，敢于挖掘问题的本质。最后，要善于交流与合作。在探求答案的过程中，与他人分享观点，取长补短，共同成长。

总之，在这个互联网和人工智能的时代，我们的问题不会越来越少，而是会越来越多。这是因为问题与答案之间存在一种辩证的关系，它们相互促进，共同推动我们探求智慧。让我们在问题的引领下，不断拓展认知边界，追求真理，成为更好的自己。
```

### AI 判分

![](https://static-main.aiyeshi.cn/images/gaokao-202406/gaokao-ai-score_button.png)

在用户点击 AI 判分按钮时，会调用`/api/gaokao/write`接口，该接口接收两个参数：

- `question`：作文的题目信息，用于大模型作文判分的依据
- `content`：作文的内容

接口的 Controller 代码如下：

```javascript
@Post('/score')
async getScore(@Body() body) {
  const { content, question } = body;

  const data = await this.gaokaoService.getScore({ content, question });

  return {
    success: !!data,
    data,
    info: !!data ? null : '生成失败',
  };
}
```

判分逻辑对应的 Service 逻辑如下：

```javascript
  // 获取判分
async getScore({ question, content }) {
  try {
    const chatCompletion: OpenAI.Chat.ChatCompletion =
      await openai.chat.completions.create({
        model: 'glm-4',
        messages: [
          {
            role: 'system',
            content: `你是一名高考的语文阅卷组老师，正在进行高考作文的批改，总分60分。你需要按照高考的作文评分标准进行给分，评分标准如下：
          \`\`\`json
          ${JSON.stringify(GAOKAO_RULE)}
          \`\`\`。
          评分标准中有不同的等级，你需要判定内容、表达、特征等方面的对应等级，并在对应等级内的最高分和最低分之间评分。评分时需考虑以下几点：
          1. 你需要先看内容的对应等级，再看表达和特征的对应等级，后面两个的等级不能高于内容的等级。
          2. 等级对应type字段，1代表最高档，4代表最低档。
          3. 每个等级的分数需要参考minScore和maxScore两个字段，你需要在这两个字段之间给出分数，对应的等级不允许高于其maxScore，也不允许低于minScore。
          4. 如果作文字数不够、偏离题意、白卷或者有严重的思想问题，你需要给出最低档的分数甚至0分。
          5. 你务必返回类似下面的格式的JSON数据：\`\`\`json
          ${JSON.stringify(GAOKAO_SOCRE_RESPONSE_EXAMPLE)}
          \`\`\`，包含score/type/rules三个字段，否则学生的考试成绩可能会受到影响，后果非常严重！
          `,
          },
          {
            role: 'user',
            content: `你是一名高考阅卷组老师，正在进行高考作文的批改，
          作文的题目的题干：{${question}}。学生对应该题干的作文内容是：{${content}}。
          请给出你评分结果的JSON数据。如果内容为空、字数不够或偏离题意，请按照最低档评分。
          请确保每个项目（内容、表达、特征）的分数在对应的等级的minScore和maxScore之间，1等不超过20分，2等不超过15分，3等不超过10分，4等不超过5分，并且总分不超过60分。
          `,
          },
        ],
        temperature: 0.2,
      });

    // 返回解析后的结果
    const data = chatCompletion?.choices[0]?.message?.content;
    const response = parseAIJSON(data) || null;

    return response;
  } catch (error) {
    return error.message;
  }
}
```

## 前端开发

![前端](https://static-main.aiyeshi.cn/images/gaokao-202406/cover-images.jpg)

前端开发部分比较简单，我们采用的是Nuxt3 + TailwindCSS + Vant的技术栈，并且基于TailwindCSS，前端也能视线很好的响应式。

数据存储部分采用的是Pinia，方便作文数据和批改数据的存储和管理。

前端主页的代码如下：

```tsx
<template>
  <div class="flex flex-col">
    <header class="px-6 py-3 flex justify-between items-center border-b">
      <div class="flex flex-col lg:flex-row lg:items-center">
        <span class="font-bold text-base md:text-xl">2024高考+AI</span>
        <span
          class="lg:ml-2 text-sm cursor-pointer underline hover:text-indigo-600 text-indigo-700"
          @click="showRules = true"
          >高考作文评分细则</span
        >
      </div>
      <div class="flex items-center">
        <span class="mr-4">{{ city }}</span>
        <van-button
          size="small"
          @click="
            () => {
              showCityPicker = true;
            }
          "
          >切换城市</van-button
        >
        <HelpButton class="relative !right-0 !top-0 ml-2 lg:ml-8" />
      </div>
    </header>
    <div class="flex flex-col p-5 lg:flex-row lg:px-8 pb-12">
      <!-- 作文题 + 作文 -->
      <div class="lg:w-1/2 lg:flex-none">
        <!-- 题目 -->
        <QuestionDesc />
        <!-- 写作 -->
        <div class="flex justify-end mt-4">
          <van-button
            type="primary"
            class="!bg-indigo-500 !border-none"
            @click="onAIWrite"
          >
            <span class="flex items-center"
              ><SparklesIcon class="w-5 mr-1" />AI写作
            </span>
          </van-button>
        </div>
        <!-- 写作的内容 -->
        <div>
          <EssayContent
            :data="content"
            @change="
              (value) => {
                content = value;
              }
            "
          />
        </div>
        <!-- AI批改 -->
        <div class="flex justify-end mt-4">
          <van-button
            type="primary"
            class="!bg-indigo-500 !border-none"
            :disabled="!content"
            @click="onAICheck"
          >
            <span class="flex items-center"
              ><SparklesIcon class="w-5 mr-1" />AI判分
            </span>
          </van-button>
        </div>
      </div>
      <!-- 批改结果 -->
      <CheckResult class="lg:w-1/2 lg:mt-0 lg:ml-8" />
    </div>

    <!-- 选择器 -->
    <van-popup v-model:show="showCityPicker" position="top">
      <van-picker title="省份" :columns="cityOptions" @confirm="onConfirm"
    /></van-popup>

    <!-- loading -->
    <van-overlay :show="loading">
      <CubeLoading class="mt-[20vh]" textCSS="text-white" :text="loadingText" />
    </van-overlay>

    <van-overlay :show="showRules" @click="showRules = false">
      <div class="pt-[15vh]">
        <div class="px-8 max-w-5xl m-auto" @click.stop>
          <img
            src="https://static-main.aiyeshi.cn/images/gaokao-202406/gaokao-zuowen-rule.jpeg"
          />
        </div>
      </div>
    </van-overlay>
  </div>
</template>

<script setup>
import { ref, toRefs, computed } from "vue";
import useProjectGaokaoStore from "@/store/gaokao";
import { SparklesIcon } from "@heroicons/vue/24/outline";

// gaokaoStore
const gaokaoStore = useProjectGaokaoStore();
const { content, cityList, city, loading, loadingText } = toRefs(gaokaoStore);

const showCityPicker = ref(false);
const showRules = ref(false);

const cityOptions = computed(() => {
  return cityList.value.map((i) => {
    return {
      text: i.city,
      value: i.city,
    };
  });
});

function onAIWrite() {
  gaokaoStore.onWrite();
}

function onAICheck() {
  gaokaoStore.onGetScore();
}

function onConfirm(city) {
  showCityPicker.value = false;
  gaokaoStore.onCityChange(city.selectedValues[0]);
}
</script>

```


核心的逻辑封装到了对应的 Store 中，`@/store/gaokao`的代码如下：
  
```javascript
import { ref, onMounted, computed } from "vue";
import { showFailToast } from "vant";
import useAxios from "@/composables/useAxios";
import cityData from "@/components/data";

/**
 * AI舌苔项目的store
 */
const useProjectGaokaoStore = defineStore("project-gaokao", () => {
  const axios = useAxios();

  const loading = ref(false);
  const loadingText = ref("正在创作中...");

  // 当前的城市
  const city = ref("新课标I卷");

  // 城市列表
  const cityList = ref(cityData);

  // 当前的内容
  const content = ref("");

  // 当前的题干
  const question = computed(() => {
    return cityList.value.find((item) => item.city === city.value).question;
  });

  // 开始进行AI写作
  async function onWrite() {
    loading.value = true;
    loadingText.value = "正在写作中...";

    result.value = null;

    try {
      const res = await axios.post("/api/gaokao/write", {
        city: city.value,
        question: question.value,
      });
      if (res.success) {
        content.value = res.data;
      } else {
        showFailToast(res.info);
      }
      loading.value = false;
    } catch (error) {
      loading.value = false;
    }
  }

  /**
  interface IResult {
    score: number;
    type: number;
    rules: {
      name: string,
      score: number,
      type: number,
      desc: string,
    }[];
  }
   */
  const result = ref(null);

  // 开始进行AI的批改
  async function onGetScore() {
    loading.value = true;
    loadingText.value = "正在批改中...";

    try {
      const res = await axios.post("/api/gaokao/score", {
        question: question.value,
        content: content.value,
      });
      if (res.success) {
        result.value = res.data;
      } else {
        showFailToast(res.info);
      }
      loading.value = false;
    } catch (error) {
      loading.value = false;
    }
  }

  function onCityChange(value) {
    city.value = value;

    cityData.forEach((item) => {
      if (item.city === value) {
        content.value = item.content;
        result.value = item.result;
      }
    });
  }

  onMounted(() => {
    // 初始化数据为北京
    onCityChange("新课标I卷");
  });

  return {
    loading,
    loadingText,
    city,
    question,
    cityList,
    content,
    result,
    onCityChange,
    onWrite,
    onGetScore,
  };
});

export default useProjectGaokaoStore;

```

  
## 最后

整个项目是一个比较简单的AI应用，没有涉及到太多AI应用的复杂逻辑，但是对于高考作文的写作和批改，是一个比较有意义（趁热点）的一个应用。

该项目的源码可以在[Github仓库](https://github.com/KapiAI/KapiAI-Apps)中获取，获取的源码可以进行商业化的二次开发，增加更加高效的Prompt和大模型的微调逻辑。


