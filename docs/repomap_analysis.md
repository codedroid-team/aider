# aider/repomap.py 分析文档

## 1. 文件概述
`aider/repomap.py` 是一个用于生成代码仓库地图(Repo Map)的工具，主要功能是分析代码库中的文件结构、函数定义和引用关系，生成结构化的代码库。

## 2. RepoMap

### 关键方法
- `get_repo_map()` - 获取代码库地图的主入口
- `get_ranked_tags()` - 获取带权重的标签(defines和references)
- `get_ranked_tags_map()` - 生成带权重的代码库地图
- `to_tree()` - 将标签转换为树形结构表示

## 3. 缓存机制
- 使用sqlite缓存
- `TAGS_CACHE_DIR` - 缓存目录
- `load_tags_cache()` - 加载缓存
- `tags_cache_error()` - 处理缓存错误


## 4. get_ranked_tags方法详解
该方法通过分析代码库中的标识符定义和引用关系，计算文件的重要性排名.方便在get_ranked_tags_map_uncached生成的树结构的token数量尽可能接近但不超出最大限制。

### 工作流程
1. **数据收集**
   - 收集所有文件中的标识符定义(`defines`)和引用(`references`)
   - 为聊天中提及的文件和标识符设置个性化权重(`personalization`)

2. **构建有向图**
   - 创建有向图(`MultiDiGraph`)，节点代表文件，边代表标识符引用关系
   - 边的权重考虑多个因素：
     * 标识符是否在聊天中被提及,被提及的标识符(`mentioned_idents`)权重×10
     * 驼峰式(`is_camel`)和蛇形(`is_snake`)通常用于重要函数/类名,并且名称长度>=8 权重×10
     * 短名或全小写名可能是临时变量，权重较低,下划线开头标识符权重×0.1(可能是私有)
     * 过多定义(>5处)权重×0.1(可能是通用名)
     * 聊天文件×50倍权重

3. **PageRank计算**
   - 使用NetworkX的PageRank算法计算文件重要性
   - (`personalization`)影响排名结果

4. **排序**
   - 将PageRank分数分配到具体的标识符定义
   - 确保聊天中未提及的重要文件被包含
   - 返回排序后的标签列表

5. **repo map dump**
   - 使用aider/aider/coders为样本生成repomap 无聊天文件 repo_max_token=1024
```
architect_prompts.py:
⋮
│class ArchitectPrompts(CoderPrompts):
⋮

ask_prompts.py:
⋮
│class AskPrompts(CoderPrompts):
⋮

base_coder.py:
⋮
│class Coder:
│    abs_fnames = None
⋮
│    def abs_root_path(self, path):
⋮
│    def get_file_mentions(self, content, ignore_current=False):
⋮
│    def get_multi_response_content_in_progress(self, final=False):
⋮
│    def get_rel_fname(self, fname):
⋮
│    def get_inchat_relative_files(self):
⋮
│    def parse_partial_args(self):
⋮

base_prompts.py:
│class CoderPrompts:
⋮

chat_chunks.py:
⋮
│@dataclass
│class ChatChunks:
│    system: List = field(default_factory=list)
⋮
│    def all_messages(self):
⋮
│    def add_cache_control(self, messages):
⋮

context_prompts.py:
⋮
│class ContextPrompts(CoderPrompts):
⋮

editblock_fenced_prompts.py:
⋮
│class EditBlockFencedPrompts(EditBlockPrompts):
⋮

editblock_func_prompts.py:
⋮
│class EditBlockFunctionPrompts(CoderPrompts):
⋮

editblock_prompts.py:
⋮
│class EditBlockPrompts(CoderPrompts):
⋮

editor_diff_fenced_prompts.py:
⋮
│class EditorDiffFencedPrompts(EditBlockFencedPrompts):
⋮

editor_editblock_prompts.py:
⋮
│class EditorEditBlockPrompts(EditBlockPrompts):
⋮

editor_whole_prompts.py:
⋮
│class EditorWholeFilePrompts(WholeFilePrompts):
⋮

help_prompts.py:
⋮
│class HelpPrompts(CoderPrompts):
⋮

patch_coder.py:
⋮
│class DiffError(ValueError):
⋮

search_replace.py:
⋮
│class RelativeIndenter:
│    """Rewrites text files to have relative indentation, which involves
│    reformatting the leading white space on lines.  This format makes
│    it easier to search and apply edits to pairs of code blocks which
│    may differ significantly in their overall level of indentation.
│
│    It removes leading white space which is shared with the preceding
│    line.
│
│    Original:
│    ```
⋮
│    def select_unique_marker(self, chars):
⋮
│    def make_absolute(self, text):
⋮
│def map_patches(texts, patches, debug):
⋮
│def relative_indent(texts):
⋮
│def lines_to_chars(lines, mapping):
⋮
│def reverse_lines(text):
⋮
│def try_strategy(texts, strategy, preproc):
⋮
│def strip_blank_lines(texts):
⋮
│def read_text(fname):
⋮
│def colorize_result(result):
⋮

shell.py:
│shell_cmd_prompt = """
⋮
│no_shell_cmd_prompt = """
│Keep in mind these details about the user's platform and environment:
│{platform}
⋮
│shell_cmd_reminder = """
│Examples of when to suggest shell commands:
│
│- If you changed a self-contained html file, suggest an OS-appropriate command to open a browser to
│- If you changed a CLI program, suggest the command to run it to see the new behavior.
│- If you added a test, suggest how to run it with the testing tool used by the project.
│- Suggest OS-appropriate commands to delete or rename files/directories, or other file system opera
│- If your code changes add new dependencies, suggest the command to install them.
│- Etc.
│
⋮

single_wholefile_func_prompts.py:
⋮
│class SingleWholeFileFunctionPrompts(CoderPrompts):
⋮

udiff_coder.py:
⋮
│def do_replace(fname, content, hunk):
⋮

udiff_prompts.py:
⋮
│class UnifiedDiffPrompts(CoderPrompts):
⋮

udiff_simple_prompts.py:
⋮
│class UnifiedDiffSimplePrompts(UnifiedDiffPrompts):
⋮

wholefile_func_prompts.py:
⋮
│class WholeFileFunctionPrompts(CoderPrompts):
⋮

wholefile_prompts.py:
⋮
│class WholeFilePrompts(CoderPrompts):
⋮
```



## 5. Kotlin移植考虑事项

将Python实现的repomap.py移植到Kotlin需要考虑以下问题：

### 1. 语法tags替代
- 使用Kotlin兼容的tree-sitter绑定`io.github.tree-sitter:ktreesitter`,ktreesitter并没有自己的解析器绑定 

### 2. 图计算库替代
- 选择JGraphT替代NetworkX
- JGraphT的pageRank没有带personalization参数构造方法.要参考`networkx/algorithms/link_analysis/pagerank_alg.py`重新实现
- nx.pagerank tagrank输出
```
shell.py: 0.349607345945613490
base_coder.py: 0.106188176902534626
search_replace.py: 0.099787660164322223
chat_chunks.py: 0.058809937726244356
base_prompts.py: 0.056984813687262943
editblock_coder.py: 0.019549370405923975
udiff_coder.py: 0.018242133136971189
patch_coder.py: 0.016027953925723683
help_prompts.py: 0.015557769052133075
editor_whole_prompts.py: 0.015148372349727768
ask_prompts.py: 0.015148372349727768
udiff_simple_prompts.py: 0.015148372349727768
wholefile_prompts.py: 0.013564172744507639
udiff_prompts.py: 0.013476316242513772
editblock_fenced_prompts.py: 0.012399867698069495
editor_editblock_prompts.py: 0.011642749124590807
editor_diff_fenced_prompts.py: 0.011642749124590807
architect_prompts.py: 0.010518972825171169
context_prompts.py: 0.010346848532248827
single_wholefile_func_prompts.py: 0.009727145480603131
wholefile_func_prompts.py: 0.009292355184581678
editblock_func_prompts.py: 0.009187377533074734
editblock_prompts.py: 0.008433266673863102
wholefile_coder.py: 0.008126773055532277
single_wholefile_func_coder.py: 0.007983933942318718
wholefile_func_coder.py: 0.007660408385140368
context_coder.py: 0.007413201234478425
editblock_func_coder.py: 0.006281937824053923
architect_coder.py: 0.006262958634619571
patch_prompts.py: 0.005970355036203351
help_coder.py: 0.005798037984103362
editor_whole_coder.py: 0.005486025728093070
ask_coder.py: 0.005486025728093070
editblock_fenced_coder.py: 0.005486025728093070
udiff_simple.py: 0.005486025728093070
editor_diff_fenced_coder.py: 0.005486025728093070
editor_editblock_coder.py: 0.005486025728093070
__init__.py: 0.005154140375263594
```
- JGraphT pagerank personalization改造后 tagrank输出
```
shell.py: 0.34898641282062937
base_coder.py: 0.10103017888384684
search_replace.py: 0.09566198649667385
chat_chunks.py: 0.059087033438952635
base_prompts.py: 0.05531734817206292
editblock_coder.py: 0.020967243544926507
help_prompts.py: 0.017541967652607852
udiff_simple_prompts.py: 0.016984161039993145
ask_prompts.py: 0.016984161039993145
editor_whole_prompts.py: 0.016984161039993145
udiff_coder.py: 0.01685889642244869
patch_coder.py: 0.013710393639764337
wholefile_prompts.py: 0.013329829078662106
udiff_prompts.py: 0.013187486860907732
architect_prompts.py: 0.011874228006314888
editor_editblock_prompts.py: 0.011765967288202773
editor_diff_fenced_prompts.py: 0.011765967288202773
editblock_fenced_prompts.py: 0.011765967288202773
context_prompts.py: 0.011630601731679492
single_wholefile_func_prompts.py: 0.011387998344623174
wholefile_func_prompts.py: 0.0106023558611432
editblock_func_prompts.py: 0.01040742125562381
single_wholefile_func_coder.py: 0.008369125003287067
editblock_prompts.py: 0.008017546079208224
wholefile_func_coder.py: 0.007992372761486214
wholefile_coder.py: 0.007766057522044365
context_coder.py: 0.007354962118177332
architect_coder.py: 0.006403852806330769
patch_prompts.py: 0.0063859572448727794
editblock_func_coder.py: 0.006162850713555287
help_coder.py: 0.005835251689208349
editblock_fenced_coder.py: 0.005458499447407497
ask_coder.py: 0.005458499447407497
editor_whole_coder.py: 0.005458499447407497
editor_editblock_coder.py: 0.005458499447407497
udiff_simple.py: 0.005458499447407497
editor_diff_fenced_coder.py: 0.005458499447407497
__init__.py: 0.005129260181930059
```
### 3. 缓存机制重设计  
- 使用Caffeine实现内存缓存

### 4. API兼容性
- 确保输出格式一致

