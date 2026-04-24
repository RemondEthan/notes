完整的 Vim IDE 环境，支持 Python、C++ 和 Rust 开发，包含以下核心功能：
1. **基础编辑功能**：行号、缩进、自动换行等基础设置
2. **文件管理**：通过 NerdTree 进行文件浏览
3. **代码补全**：基于 CoC (Conquer of Completion) 提供智能补全
4. **语法检查**：使用 ALE 进行实时语法检查
5. **代码格式化**：支持自动格式化
6. **版本控制**：集成 Git 功能
7. **语言特定支持**：针对 Python、C++ 和 Rust 的专用插件和配置
### 常用快捷键
- `Ctrl+n`：打开 / 关闭文件浏览器
- `Ctrl+h/j/k/l`：窗口间切换
- `Ctrl+s`：保存文件
- `gd`：跳转到定义
- `gr`：查看引用
- `leader+rn`：重命名符号
- `leader+f`：格式化代码
- `space`：代码折叠 / 展开
- `Ctrl+c`:向左切换文件标签
- `Ctrl+v`:向右切换
- `gd`:goto define
- `gr`: goto reference
- `Ctrl+o`:回退到前一个鼠标代码出
- `Ctrl+i`:反向回退
- `m`:在文件浏览窗口打开菜单

>安装过程：
>Ubuntu/Debian
	sudo apt install vim git curl python3 python3-pip clang rustc cargo flake8 pylint
macOS (使用Homebrew)
	brew install vim git curl python clang rust flake8 pylint

我这里遇到一些问题，所以讲node重装了，指定了版本18
```bash
# 卸载现有 Node.js（若有）
brew uninstall node

# 安装 LTS 版本（推荐 18.x 或 20.x）
brew install node@18

# 链接到系统路径
brew link --force node@18

#在执行link的时候会有提示，按照系统提示进行操作，brew link --force会因为一些目录存在而失败,到最后也没有`/usr/lib/libc++.1.dylib`，这个文件，感觉似乎不影响使用，因为提示要系统更新，我这是黑苹果，算了，不要更新了
```
```vim
:CocUninstall coc-rls
# 因为rls是老版本的组件，新版本统一到rust-analyzer，安装过程中还有rls，所以卸载一下
```
```bash
#直接上大招，用vscode的版本，这里是我从github下载的新版本，这个链接制作参考就好了，下载到~/.local/bin下，在vimrc文件里进行了配置
# 下载 VS Code 内置的 rust-analyzer（版本稳定）
curl -L https://github.com/rust-lang/rust-analyzer/releases/download/2024-05-13/rust-analyzer-aarch64-apple-darwin.gz -o rust-analyzer.gz
# 解压并移动到 PATH 路径
gunzip rust-analyzer.gz
chmod +x rust-analyzer
mv rust-analyzer ~/.local/bin/
```

~~~vimrc
" ==============================================
" 基础配置
" ==============================================
set nocompatible              " 禁用VI兼容性
filetype off                  " 关闭文件类型检测(稍后重新开启)
set encoding=utf-8            " 使用UTF-8编码
set number                    " 显示行号
set norelativenumber            " 显示相对行号
set cursorline                " 高亮当前行
set mouse=a                   " 启用鼠标支持
set clipboard=unnamedplus     " 共享剪贴板
set autoindent                " 自动缩进
set smartindent               " 智能缩进
set tabstop=4                 " Tab键宽度
set shiftwidth=4              " 自动缩进宽度
set expandtab                 " 将Tab转换为空格
set smarttab                  " 智能Tab处理
set wrap                      " 自动换行
set linebreak                 " 按单词换行
set scrolloff=5               " 光标上下保留5行
set showcmd                   " 显示命令
set laststatus=2              " 总是显示状态栏
set statusline=%F%m%r%h%w\ [FORMAT=%{&ff}]\ [TYPE=%Y]\ [POS=%l,%v][%p%%]\ %{strftime(\"%d/%m/%Y\ -\ %H:%M\")}
set backup                    " 启用备份
set backupdir=~/.vim/backup// " 备份文件目录
set undodir=~/.vim/undo//     " 撤销历史目录
set undofile                  " 保存撤销历史
set ignorecase                " 搜索忽略大小写
set smartcase                 " 智能大小写搜索
set incsearch                 " 增量搜索
set hlsearch                  " 高亮搜索结果
set wildmenu                  " 命令行补全菜单
set wildmode=longest,full     " 命令行补全模式
let mapleader = ";"
set mouse-=v  " 移除 Visual 模式的鼠标支持
set selectmode=  " 取消所有选择模式（鼠标和键盘均不触发 Visual 模式）

" 创建备份和撤销目录
if !isdirectory(expand('~/.vim/backup'))
    call mkdir(expand('~/.vim/backup'), 'p')
endif
if !isdirectory(expand('~/.vim/undo'))
    call mkdir(expand('~/.vim/undo'), 'p')
endif

" ==============================================
" 插件管理器: vim-plug
" ==============================================
let data_dir = has('nvim') ? stdpath('data') . '/site' : '~/.vim'
if empty(glob(data_dir . '/autoload/plug.vim'))
  silent execute '!curl -fLo '.data_dir.'/autoload/plug.vim --create-dirs  https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
  autocmd VimEnter * PlugInstall --sync | source $MYVIMRC
endif

call plug#begin()

" ==============================================
" 通用开发插件
" ==============================================
Plug 'tpope/vim-sensible'                " 基础配置增强
Plug 'tpope/vim-surround'                " 环绕操作
Plug 'tpope/vim-commentary'              " 注释插件
Plug 'tpope/vim-fugitive'                " Git集成
Plug 'junegunn/gv.vim'                   " Git提交历史查看
Plug 'airblade/vim-gitgutter'            " Git变更提示
Plug 'preservim/nerdtree'                " 文件浏览器
Plug 'Xuyuanp/nerdtree-git-plugin'       " NerdTree的Git支持
Plug 'ryanoasis/vim-devicons'            " 图标支持
Plug 'vim-airline/vim-airline'           " 状态栏增强
Plug 'vim-airline/vim-airline-themes'    " 状态栏主题
Plug 'junegunn/fzf', { 'do': { -> fzf#install() } } " 模糊搜索
Plug 'junegunn/fzf.vim'                  " FZF的Vim集成
Plug 'Yggdroot/indentLine'               " 显示缩进线
Plug 'mhinz/vim-startify'                " 启动页面
Plug 'tpope/vim-unimpaired'              " 便捷操作快捷键
Plug 'morhetz/gruvbox'
Plug 'joshdick/onedark.vim'
" ==============================================
" 代码补全与LSP
" ==============================================
Plug 'neoclide/coc.nvim', {'branch': 'release'} " 智能补全框架

" ==============================================
" 语法高亮与检查
" ==============================================
Plug 'sheerun/vim-polyglot'              " 多语言语法高亮
Plug 'dense-analysis/ale'                " 异步语法检查

" ==============================================
" 代码格式化
" ==============================================
Plug 'sbdchd/neoformat'                  " 代码格式化

" ==============================================
" Python 开发
" ==============================================
Plug 'python-mode/python-mode'           " Python开发增强
Plug 'davidhalter/jedi-vim'              " Python自动补全
Plug 'tmhedberg/SimpylFold'              " Python代码折叠

" ==============================================
" C/C++ 开发
" ==============================================
Plug 'octol/vim-cpp-enhanced-highlight'  " C++语法高亮增强
Plug 'jackguo380/vim-lsp-cxx-highlight'  " C++ LSP高亮
Plug 'rhysd/vim-clang-format'            " Clang格式化

" ==============================================
" Rust 开发
" ==============================================
Plug 'rust-lang/rust.vim'                " Rust语法高亮
Plug 'racer-rust/vim-racer'              " Rust自动补全
Plug 'timonv/vim-cargo'                  " Cargo集成

call plug#end()

" ==============================================
" 插件配置
" ==============================================

" NerdTree配置
map <C-n> :NERDTreeToggle<CR>             " Ctrl+n 打开/关闭NERDTree
autocmd bufenter * if (winnr("$") == 1 && exists("b:NERDTree") && b:NERDTree.isTabTree()) | q | endif
let g:NERDTreeShowHidden=1                " 显示隐藏文件
let g:NERDTreeIgnore=['\.pyc$', '\~$']    " 忽略的文件类型
let g:NERDTreeMinimalUI=1                 " 简化UI
let g:NERDTreeStatusline=0                " 不显示状态栏

" Airline配置
let g:airline_powerline_fonts = 1         " 使用Powerline字体
let g:airline_theme = 'papercolor'        " 设置主题
let g:airline#extensions#tabline#enabled = 1 " 启用标签页

" ALE配置
let g:ale_lint_on_text_changed = 'always' " 文本改变时检查
let g:ale_lint_on_insert_leave = 1        " 离开插入模式时检查
let g:ale_sign_error = '✗'                " 错误符号
let g:ale_sign_warning = '⚠'              " 警告符号

" 对不同语言使用不同的linter
let g:ale_linters = {
    \ 'python': ['flake8', 'pylint'],
    \ 'cpp': ['clang', 'cppcheck'],
    \ 'rust': ['rustc', 'cargo'],
    \}

" Neoformat配置
let g:neoformat_try_node_exe = 1

" Python-Mode配置
let g:pymode_python = 'python3'           " 使用Python3
let g:pymode_lint_on_save = 1             " 保存时检查
let g:pymode_lint_checkers = ['flake8']   " 使用flake8检查
let g:pymode_rope = 1                     " 启用rope自动补全

" SimpylFold配置
let g:SimpylFold_docstring_preview = 1    " 显示文档字符串预览

" Rust配置
let g:rustfmt_autosave = 1                " 保存时自动格式化
"let g:racer_cmd = "$HOME/.cargo/bin/racer" " Racer路径
"let g:racer_autostart = 1                 " 自动启动Racer

" Clang-format配置
let g:clang_format#style_options = {
    \ "BasedOnStyle": "Google",
    \ "IndentWidth": 4,
    \ "SortIncludes": "false"
    \}
let g:coc_rust_analyzer_path = '/Users/ksw/.local/bin/rust-analyer'
" ==============================================
" CoC配置 (智能补全)
" ==============================================
" 显示签名帮助
autocmd CursorHoldI * silent! call CocActionAsync('showSignatureHelp')

" 代码补全
inoremap <silent><expr> <TAB>
      \ pumvisible() ? "\<C-n>" :
      \ <SID>check_back_space() ? "\<TAB>" :
      \ coc#refresh()
inoremap <expr><S-TAB> pumvisible() ? "\<C-p>" : "\<C-h>"

" 跳转到定义
nmap <silent> gd <Plug>(coc-definition)
" 跳转到声明
nmap <silent> gD <Plug>(coc-declaration)
" 跳转到实现
nmap <silent> gi <Plug>(coc-implementation)
" 跳转到引用
nmap <silent> gr <Plug>(coc-references)

" 重命名
nmap <leader>rn <Plug>(coc-rename)

" 格式化选中区域
xmap <leader>f  <Plug>(coc-format-selected)
nmap <leader>f  <Plug>(coc-format-selected)

" 应用代码动作
nmap <leader>ac  <Plug>(coc-codeaction)

" 启用诊断
nnoremap <silent> <space>e :<C-u>CocList diagnostics<CR>

" 设置CoC扩展
let g:coc_global_extensions = [
    \ 'coc-pyright',
    \ 'coc-clangd',
    \ 'coc-rust-analyzer',
    \ 'coc-json',
    \ 'coc-yaml',
    \ 'coc-vimlsp',
    \ 'coc-tsserver',
    \ 'coc-prettier',
    \ 'coc-git',
    \ 'coc-snippets',
    \ 'coc-marketplace'
    \]

" ==============================================
" 快捷键配置
" ==============================================
" 窗口切换
nnoremap <C-h> <C-w>h
nnoremap <C-j> <C-w>j
nnoremap <C-k> <C-w>k
nnoremap <C-l> <C-w>l

" 快速保存
nnoremap <C-s> :w<CR>
inoremap <C-s> <ESC>:w<CR>a

" 快速关闭当前窗口
nnoremap <C-q> :q<CR>

" 快速切换标签页
"nnoremap <C-Left> :tabprevious<CR>
"nnoremap <C-Right> :tabnext<CR>
nnoremap <C-c> :bprevious<CR>
nnoremap <C-v> :bnext<CR>
nnoremap <C-d> :bd<CR>  " bd = buffer delete

" 代码折叠
nnoremap <space> za

" 格式化代码
nnoremap <leader>fm :Neoformat<CR>

" ==============================================
" 自动命令
" ==============================================
" 文件类型检测
filetype plugin indent on

" 针对不同文件类型的设置
autocmd FileType python setlocal shiftwidth=4 tabstop=4 softtabstop=4
autocmd FileType cpp setlocal shiftwidth=4 tabstop=4 softtabstop=4
autocmd FileType rust setlocal shiftwidth=4 tabstop=4 softtabstop=4

" 自动移除行尾空格
autocmd BufWritePre * %s/\s\+$//e

" Python自动添加# -*- coding: utf-8 -*-
autocmd BufNewFile *.py 0r ~/.vim/templates/python_header
autocmd BufNewFile *.py normal! G

" C++自动添加头文件模板
autocmd BufNewFile *.cpp 0r ~/.vim/templates/cpp_header
autocmd BufNewFile *.h 0r ~/.vim/templates/h_header

" Rust自动添加模板
autocmd BufNewFile *.rs 0r ~/.vim/templates/rust_header

" ==============================================
" 配色方案
" ==============================================
" set termguicolors                  " 启用真彩色
set background=dark  " 或 light 切换明暗模式
colorscheme gruvbox
~~~
