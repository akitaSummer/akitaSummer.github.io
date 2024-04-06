---
title: 编译原理实战——实现一个简单的js引擎
date: 2023-11-12 03:36:40
tags:
  - javascript engine
  - rust
  - 编译原理
categories: 学习笔记
---

# 前言

之前我们实现了一个简单的浏览器，但是其中最为核心的 js 引擎，我们使用的是 v8，并没有自己实现，本次我们将会实现一个简单的 js 引擎，作为我们编译原理学习的一个练习。

# 设计

js 引擎，实际上是一个编译器，编译器分为前端和后端两个部分，前端的主要作用是将源代码编译成中间代码，她会包含词法分析和语法分析，而后端则是将中间代码优化并产生最终代码（如机器码）。

词法解析就是 tokenize，将源码切成一个个 token。语法分析就是要生成中间码，比如 ast。

后端一般会做一下中间代码优化，比如对常量替换，中间内联，死代码提出等等，这是工业级编译器中最难，最重要的部分，不过我们只是实现一个编译器，这个部分我们并不会去编写。

最后就是代码生成，我们这次会设计一个 vm，我们最终代码就是 vm 的字节码。

# 词法解析

## token 设计

在第一步，我们需要先明确我们的 token，在这里我分为了两个部分，字面量（boolean, null...），Identifier(变量名、方法名、类名、参数名等)，Keyword 和 Symbol，首先我们先实现 Keyword 和 Symbol，由于他们在语法分析时会起到标志性作用，所以需要分别进行存储他们对应的映射，用于后续语法分析时查找优化：

```rust

// 参考[acorn](https://github.com/acornjs/acorn/blob/master/acorn/src/tokentype.js)
// 通过为每种标记类型附加一个 beforeExpr 属性，来指示在这些标记之后的斜杠是否应该被视为正则表达式的开始。如果它们的 beforeExpr 属性的值为 true，那么在词法分析阶段，它可以遵循正则表达式的定义来生成正则表达式字面量标记。这个方法的目的是在词法分析阶段正确地处理JavaScript正则表达式，确保正则表达式的语法与整个JavaScript语法相协调。
// 生成4个集合，分别是name set, key:name map, name:key map, key set, 用于关键词
macro_rules! gen_map {
    ($($k:expr => $v:expr, $be:expr)*) => {
      {
        let mut s = HashSet::new();
        $(
          s.insert($v);
        )*
        let mut kv = HashMap::new();
        $(
          kv.insert($k, $v);
        )*
        let mut vk = HashMap::new();
        $(
          vk.insert($v, $k);
        )*
        let mut be = HashSet::new();
        $(
          be.insert($k);
        )*
        (Some(s), Some(kv), Some(vk), Some(be))
      }
    };
  }

#[derive(Debug, Eq, PartialEq, Hash, Copy, Clone)]
pub enum Keyword {
    Break,
    Do,
    Instanceof,
    Typeof,
    Case,
    Else,
    New,
    Var,
    Catch,
    Finally,
    Return,
    Void,
    Continue,
    For,
    Switch,
    While,
    Debugger,
    Function,
    This,
    With,
    Default,
    If,
    Throw,
    Delete,
    In,
    Try,
    // 暂不支持
    Class,
    Enum,
    Extends,
    Super,
    Const,
    Export,
    Import,
}

static mut KEYWORDS_SET: Option<HashSet<&'static str>> = None;
static mut KEYWORDS_KEY_NAME: Option<HashMap<Keyword, &'static str>> = None;
static mut KEYWORDS_NAME_KEY: Option<HashMap<&'static str, Keyword>> = None;
static mut KEYWORDS_BEFORE_EXPR_SET: Option<HashSet<Keyword>> = None;
fn init_keywords() {
    let (s, kv, vk, be) = gen_map! {
      Keyword::Break => "break", false
      Keyword::Do => "do", true
      Keyword::Instanceof => "instanceof", true
      Keyword::Typeof => "typeof", true
      Keyword::Case => "case", true
      Keyword::Else => "else", true
      Keyword::New => "new", true
      Keyword::Var => "var", false
      Keyword::Catch => "catch", false
      Keyword::Finally => "finally", false
      Keyword::Return => "return", true
      Keyword::Void => "void", true
      Keyword::Continue => "continue", false
      Keyword::For => "for", false
      Keyword::Switch => "switch", false
      Keyword::While => "while", false
      Keyword::Debugger => "debugger", false
      Keyword::Function => "function", false
      Keyword::This => "this", false
      Keyword::With => "with", false
      Keyword::Default => "default", true
      Keyword::If => "if", false
      Keyword::Throw => "throw", true
      Keyword::Delete => "delete", true
      Keyword::In => "in", true
      Keyword::Try => "try", false
      // future reserved words
      Keyword::Class => "class", false
      Keyword::Enum => "enum", false
      Keyword::Extends => "extends", true
      Keyword::Super => "super", false
      Keyword::Const => "const", false
      Keyword::Export => "export", false
      Keyword::Import => "import", false
    };
    unsafe {
        KEYWORDS_SET = s;
        KEYWORDS_KEY_NAME = kv;
        KEYWORDS_NAME_KEY = vk;
        KEYWORDS_BEFORE_EXPR_SET = be;
    }
}
pub fn is_keyword(s: &str) -> bool {
    unsafe { KEYWORDS_SET.as_ref().unwrap().contains(s) }
}
pub fn keyword_to_name(v: &Keyword) -> &'static str {
    unsafe { KEYWORDS_KEY_NAME.as_ref().unwrap().get(v).unwrap() }
}
pub fn name_to_keyword(s: &str) -> Keyword {
    unsafe {
        KEYWORDS_NAME_KEY
            .as_ref()
            .unwrap()
            .get(s)
            .unwrap()
            .to_owned()
    }
}
impl Keyword {
    pub fn name(&self) -> &'static str {
        keyword_to_name(self)
    }

    pub fn is_before_expr(&self) -> bool {
        unsafe { KEYWORDS_BEFORE_EXPR_SET.as_ref().unwrap().contains(self) }
    }
}



#[derive(Debug, Eq, PartialEq, Hash, Copy, Clone)]
pub enum Symbol {
    BraceL,
    BraceR,
    ParenL,
    ParenR,
    BracketL,
    BracketR,
    Dot,
    Semi,
    Comma,
    BinOpStart,
    LT,
    GT,
    LE,
    GE,
    Eq,
    NotEq,
    EqStrict,
    NotEqStrict,
    Add,
    Sub,
    Mul,
    Div,
    Mod,
    Inc,
    Dec,
    SHL,
    SAR,
    SHR,
    BitAnd,
    BitOr,
    BitXor,
    Not,
    BitNot,
    And,
    Or,
    BinOpEnd,
    Conditional,
    Colon,
    AssignStart,
    Assign,
    AssignAdd,
    AssignSub,
    AssignMul,
    AssignDiv,
    AssignMod,
    AssignSHL,
    AssignSAR,
    AssignSHR,
    AssignBitAnd,
    AssignBitOr,
    AssignBitXor,
    AssignEnd,
}

static mut SYMBOLS_SET: Option<HashSet<&'static str>> = None;
static mut SYMBOLS_KEY_NAME: Option<HashMap<Symbol, &'static str>> = None;
static mut SYMBOLS_NAME_KEY: Option<HashMap<&'static str, Symbol>> = None;
static mut SYMBOLS_BEFORE_EXPR_SET: Option<HashSet<Symbol>> = None;
static mut SYMBOLS_KEY_PRECEDENCE: Option<HashMap<Symbol, i32>> = None;
fn init_symbols() {
    let (s, kv, vk, be, pcd) = gen_map_syb! {
      Symbol::BraceL => "{", true, 0
      Symbol::BraceR => "}", false, 0
      Symbol::ParenL => "(", true, 20
      Symbol::ParenR => ")", false, 0
      Symbol::BracketL => "[", true, 19
      Symbol::BracketR => "]", false, 0
      Symbol::Dot => ".", true, 19
      Symbol::Semi => ";", true, 0
      Symbol::Comma => ",", true, 1
      Symbol::LT => "<", true, 11
      Symbol::GT => ">", true, 11
      Symbol::LE => "<=", true, 11
      Symbol::GE => ">=", true, 11
      Symbol::Eq => "==", true, 10
      Symbol::NotEq => "!=", true, 10
      Symbol::EqStrict => "===", true, 10
      Symbol::NotEqStrict => "!==", true, 10
      Symbol::Add => "+", true, 13
      Symbol::Sub => "-", true, 13
      Symbol::Mul => "*", true, 14
      Symbol::Div => "/", true, 14
      Symbol::Mod => "%", true, 14
      Symbol::Inc => "++", false, 16
      Symbol::Dec => "--", false, 16
      Symbol::SHL => "<<", true, 12
      Symbol::SAR => ">>", true, 12
      Symbol::SHR => ">>>", true, 12
      Symbol::BitAnd => "&", true, 9
      Symbol::BitOr => "|", true, 7
      Symbol::BitXor => "^", true, 8
      Symbol::Not => "!", true, 16
      Symbol::BitNot => "~", true, 16
      Symbol::And => "&&", true, 6
      Symbol::Or => "||", true, 5
      Symbol::Conditional => "?", true, 4
      Symbol::Colon => ":", true, 4
      Symbol::Assign => "=", true, 3
      Symbol::AssignAdd => "+=", true, 3
      Symbol::AssignSub => "-=", true, 3
      Symbol::AssignMul => "*=", true, 3
      Symbol::AssignDiv => "/=", true, 3
      Symbol::AssignMod => "%=", true, 3
      Symbol::AssignSHL => "<<=", true, 3
      Symbol::AssignSAR => ">>=", true, 3
      Symbol::AssignSHR => ">>>=", true, 3
      Symbol::AssignBitAnd => "&=", true, 3
      Symbol::AssignBitOr => "|=", true, 3
      Symbol::AssignBitXor => "^=", true, 3
    };
    unsafe {
        SYMBOLS_SET = s;
        SYMBOLS_KEY_NAME = kv;
        SYMBOLS_NAME_KEY = vk;
        SYMBOLS_BEFORE_EXPR_SET = be;
        SYMBOLS_KEY_PRECEDENCE = pcd;
    }
}
pub fn is_symbol(s: &str) -> bool {
    unsafe { SYMBOLS_SET.as_ref().unwrap().contains(s) }
}
pub fn symbol_to_name(v: &Symbol) -> &'static str {
    unsafe { SYMBOLS_KEY_NAME.as_ref().unwrap().get(v).unwrap() }
}
pub fn symbol_pcd(v: &Symbol) -> i32 {
    unsafe {
        SYMBOLS_KEY_PRECEDENCE
            .as_ref()
            .unwrap()
            .get(v)
            .unwrap()
            .to_owned()
    }
}
pub fn name_to_symbol(s: &str) -> Symbol {
    unsafe {
        SYMBOLS_NAME_KEY
            .as_ref()
            .unwrap()
            .get(s)
            .unwrap()
            .to_owned()
    }
}
impl Symbol {
    pub fn name(&self) -> &'static str {
        symbol_to_name(self)
    }

    pub fn is_before_expr(&self) -> bool {
        unsafe { SYMBOLS_BEFORE_EXPR_SET.as_ref().unwrap().contains(self) }
    }
}


#[derive(Debug, Copy, Clone)]
pub struct Position {
    pub line: i32,
    pub column: i32,
}

impl Position {
    pub fn new() -> Position {
        Position { line: 0, column: 0 }
    }
}

#[derive(Debug, Copy, Clone)]
pub struct SourceLoc {
    pub start: Position,
    pub end: Position,
}

impl SourceLoc {
    pub fn new() -> Self {
        SourceLoc {
            start: Position::new(),
            end: Position::new(),
        }
    }
}

#[derive(Debug, Copy, Clone)]
pub struct KeywordData {
    pub kind: Keyword,
    pub loc: SourceLoc,
}

#[derive(Debug, Clone)]
pub struct SymbolData {
    pub kind: Symbol,
    pub loc: SourceLoc,
}

```

接下来是字面量（boolean, null...）和 Identifier，他们就比较简单：

```rust

// 标识符(变量名、方法名、类名、参数名等)

#[derive(Debug, Clone)]
pub struct IdentifierData {
    pub value: String,
    pub loc: SourceLoc,
}

// 字面量，用于描述不同数据类型的值

#[derive(Debug, Clone)]
pub struct NullLiteralData {
    pub loc: SourceLoc,
}
pub fn is_null(s: &str) -> bool {
    s == "null"
}

#[derive(Debug, Eq, PartialEq, Clone)]
pub enum BooleanLiteral {
    True,
    False,
}

#[derive(Debug, Clone)]
pub struct BooleanLiteralData {
    pub kind: BooleanLiteral,
    pub loc: SourceLoc,
}
pub fn is_bool(s: &str) -> bool {
    s == "true" || s == "false"
}
pub fn name_to_bool(s: &str) -> BooleanLiteral {
    match s {
        "true" => BooleanLiteral::True,
        "false" => BooleanLiteral::False,
        _ => panic!(),
    }
}
impl BooleanLiteral {
    pub fn name(&self) -> &'static str {
        match self {
            BooleanLiteral::True => "true",
            BooleanLiteral::False => "false",
        }
    }
}

#[derive(Debug, Clone)]
pub struct StringLiteralData {
    pub value: String,
    pub loc: SourceLoc,
}

#[derive(Debug, Clone)]
pub struct NumericLiteralData {
    pub value: String,
    pub loc: SourceLoc,
}

#[derive(Debug, Clone)]
pub struct RegExpLiteralData {
    pub value: String,
    pub loc: SourceLoc,
}

#[derive(Debug, Clone)]
pub struct EofData {
    pub loc: SourceLoc,
}

```

最后我们定义我们的 token：

```rust

// token
#[derive(Debug, Clone)]
pub enum Token {
    Keyword(KeywordData),
    Symbol(SymbolData),
    ContextualKeyword(CtxKeywordData),
    Identifier(IdentifierData),
    NullLiteral(NullLiteralData),
    BooleanLiteral(BooleanLiteralData),
    StringLiteral(StringLiteralData),
    NumericLiteral(NumericLiteralData),
    RegExpLiteral(RegExpLiteralData),
    Eof(EofData),
    Nil,
}

impl Token {
    pub fn is_keyword(&self) -> bool {
        match self {
            Token::Keyword(_) => true,
            _ => false,
        }
    }

    pub fn is_keyword_kind(&self, k: Keyword) -> bool {
        match self {
            Token::Keyword(data) => data.kind == k,
            _ => false,
        }
    }

    pub fn is_keyword_kind_in(&self, ks: &Vec<Keyword>) -> bool {
        match self {
            Token::Keyword(d) => ks.contains(&d.kind),
            _ => false,
        }
    }

    pub fn is_keyword_bin(&self, not_in: bool) -> bool {
        match self {
            Token::Keyword(d) => !not_in && d.kind == Keyword::In,
            _ => false,
        }
    }

    pub fn keyword_pcd(&self) -> i32 {
        match self {
            Token::Keyword(d) => match d.kind {
                Keyword::In => 11,
                _ => -1,
            },
            _ => -1,
        }
    }

    pub fn keyword_data(&self) -> &KeywordData {
        match self {
            Token::Keyword(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_symbol(&self) -> bool {
        match self {
            Token::Symbol(_) => true,
            _ => false,
        }
    }

    pub fn symbol_pcd(&self) -> i32 {
        match self {
            Token::Symbol(s) => symbol_pcd(&s.kind),
            _ => panic!(),
        }
    }

    pub fn is_symbol_bin(&self) -> bool {
        match self {
            Token::Symbol(s) => {
                let s = s.kind as i32;
                let start = Symbol::BinOpStart as i32;
                let end = Symbol::BinOpEnd as i32;
                s > start && s < end
            }
            _ => false,
        }
    }

    pub fn is_symbol_assign(&self) -> bool {
        match self {
            Token::Symbol(s) => {
                let s = s.kind as i32;
                let start = Symbol::AssignStart as i32;
                let end = Symbol::AssignEnd as i32;
                s > start && s < end
            }
            _ => false,
        }
    }

    pub fn is_symbol_kind(&self, k: Symbol) -> bool {
        match self {
            Token::Symbol(s) => s.kind == k,
            _ => false,
        }
    }

    pub fn is_symbol_kind_in(&self, ks: &Vec<Symbol>) -> bool {
        match self {
            Token::Symbol(d) => ks.contains(&d.kind),
            _ => false,
        }
    }

    pub fn symbol_data(&self) -> &SymbolData {
        match self {
            Token::Symbol(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_ctx_keyword(&self) -> bool {
        match self {
            Token::ContextualKeyword(_) => true,
            _ => false,
        }
    }

    pub fn ctx_keyword_data(&self) -> &CtxKeywordData {
        match self {
            Token::ContextualKeyword(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_id(&self) -> bool {
        match self {
            Token::Identifier(v) => !v.value.eq("undefined"),
            _ => false,
        }
    }

    pub fn id_data(&self) -> &IdentifierData {
        match self {
            Token::Identifier(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_undef(&self) -> bool {
        match self {
            Token::Identifier(v) => v.value.eq("undefined"),
            _ => false,
        }
    }

    pub fn undef_data(&self) -> &IdentifierData {
        match self {
            Token::Identifier(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_null(&self) -> bool {
        match self {
            Token::NullLiteral(_) => true,
            _ => false,
        }
    }

    pub fn null_data(&self) -> &NullLiteralData {
        match self {
            Token::NullLiteral(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_bool(&self) -> bool {
        match self {
            Token::BooleanLiteral(_) => true,
            _ => false,
        }
    }

    pub fn bool_data(&self) -> &BooleanLiteralData {
        match self {
            Token::BooleanLiteral(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_str(&self) -> bool {
        match self {
            Token::StringLiteral(_) => true,
            _ => false,
        }
    }

    pub fn str_data(&self) -> &StringLiteralData {
        match self {
            Token::StringLiteral(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_num(&self) -> bool {
        match self {
            Token::NumericLiteral(_) => true,
            _ => false,
        }
    }

    pub fn num_data(&self) -> &NumericLiteralData {
        match self {
            Token::NumericLiteral(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_regexp(&self) -> bool {
        match self {
            Token::RegExpLiteral(_) => true,
            _ => false,
        }
    }

    pub fn regexp_data(&self) -> &RegExpLiteralData {
        match self {
            Token::RegExpLiteral(data) => data,
            _ => panic!(),
        }
    }

    pub fn is_before_expr(&self) -> bool {
        match self {
            Token::Keyword(data) => data.kind.is_before_expr(),
            Token::Symbol(data) => data.kind.is_before_expr(),
            Token::ContextualKeyword(data) => data.kind.is_before_expr(),
            _ => false,
        }
    }

    pub fn is_eof(&self) -> bool {
        match self {
            Token::Eof(_) => true,
            _ => false,
        }
    }

    pub fn loc(&self) -> &SourceLoc {
        match self {
            Token::Keyword(data) => &data.loc,
            Token::Symbol(data) => &data.loc,
            Token::ContextualKeyword(data) => &data.loc,
            Token::Identifier(data) => &data.loc,
            Token::NullLiteral(data) => &data.loc,
            Token::BooleanLiteral(data) => &data.loc,
            Token::StringLiteral(data) => &data.loc,
            Token::NumericLiteral(data) => &data.loc,
            Token::RegExpLiteral(data) => &data.loc,
            Token::Eof(data) => &data.loc,
            Token::Nil => panic!(),
        }
    }
}

static INIT_TOKEN_DATA_ONCE: Once = Once::new();
pub fn init_token_data() {
    INIT_TOKEN_DATA_ONCE.call_once(|| {
        init_keywords();
        init_symbols();
        init_ctx_keyword();
    });
}
```

## 字符流处理

在编写词法分析器之前，我们需要有个对字符流处理的对象，他主要的目的是帮助我们词法分析器更好的获取字符：

```rust
// 处理源代码的字符流
pub struct Source<'a> {
    pub chs: Chars<'a>,
    pub peeked: VecDeque<char>,
    pub line: i32,
    pub column: i32,
}

pub const EOL: char = '\n';

// '\n'、 '\r'、 U+2028 or U+2029
pub fn is_line_terminator(c: char) -> bool {
    let cc = c as u32;
    cc == 0x0a || cc == 0x0d || cc == 0x2028 || cc == 0x2029
}

impl<'a> Source<'a> {
    pub fn new(code: &'a String) -> Self {
        Source {
            chs: code.chars(),
            peeked: VecDeque::with_capacity(3),
            line: 1,
            column: 0,
        }
    }

    // 用于从字符流中获取下一个字符
    fn next_join_crlf(&mut self) -> Option<char> {
        match self.chs.next() {
            Some(c) => {
                if is_line_terminator(c) {
                    // 如果是回车符 '\r'，则进一步检查下一个字符
                    if c == '\r' {
                        // 如果下一个字符是换行符 '\n'，则跳过回车符 '\r'，并将换行符 '\n' 存储在 peeked 队列中
                        if let Some(c) = self.chs.next() {
                            // 如果下一个字符不是换行符 '\n'，则将下一个字符存储在 peeked 队列中，以便后续读取
                            if c != '\n' {
                                self.peeked.push_back(c);
                            }
                        }
                    }
                    Some(EOL)
                } else {
                    Some(c)
                }
            }
            _ => None,
        }
    }
    // 用于读取下一个字符，优先从 peeked 中读取
    pub fn read(&mut self) -> Option<char> {
        let c = match self.peeked.pop_front() {
            Some(c) => Some(c),
            _ => self.next_join_crlf(),
        };
        if let Some(c) = c {
            if c == EOL {
                self.line += 1;
                self.column = 0;
            } else {
                self.column += 1;
            }
        }
        c
    }

    // 用于预览下一个字符，如果 peeked 非空，则返回队列头部字符，否则通过 next_join_crlf 预读取字符，并将其存储在 peeked 中
    pub fn peek(&mut self) -> Option<char> {
        match self.peeked.front().cloned() {
            Some(c) => Some(c),
            _ => match self.next_join_crlf() {
                Some(c) => {
                    self.peeked.push_back(c);
                    Some(c)
                }
                _ => None,
            },
        }
    }

    // 测试预读的字符是否与指定的字符或字符序列匹配
    pub fn test_ahead(&mut self, ch: char) -> bool {
        match self.peek() {
            Some(c) => c == ch,
            _ => false,
        }
    }

    pub fn test_ahead_or(&mut self, c1: char, c2: char) -> bool {
        match self.peek() {
            Some(c) => c == c1 || c == c2,
            _ => false,
        }
    }

    pub fn test_ahead_chs(&mut self, chs: &[char]) -> bool {
        let mut pass = true;
        for i in 0..self.peeked.len() {
            pass = match self.peeked.get(i) {
                Some(c) => *c == chs[i],
                _ => false,
            };
            if !pass {
                return false;
            }
        }
        for i in self.peeked.len()..chs.len() {
            pass = match self.next_join_crlf() {
                Some(c) => {
                    self.peeked.push_back(c);
                    c == chs[i]
                }
                _ => false,
            };
            if !pass {
                return false;
            }
        }
        pass
    }

    pub fn test_ahead2(&mut self, c1: char, c2: char) -> bool {
        self.test_ahead_chs(&[c1, c2])
    }

    pub fn test_ahead3(&mut self, c1: char, c2: char, c3: char) -> bool {
        self.test_ahead_chs(&[c1, c2, c3])
    }

    // 前进到下一个字符
    pub fn advance(&mut self) {
        self.read();
    }

    pub fn advance2(&mut self) {
        self.read();
        self.read();
    }
}
```

## lexer

在有了 token 定义及字符流处理器后，我们就可以实现我们的 lexer，核心的原理就是不断地从字符流中获取字符，判断其每个开头与对应的哪种 token 开头相同，找到相同的 token 类型，就一直解析到对应 token 的结束：

```rust

// 判断字符是否是空白字符，排除行终止符。
fn is_whitespace(c: char) -> bool {
    !is_line_terminator(c) && c.is_whitespace()
}

// 判断一个字符是否是 Unicode 字母
fn is_unicode_letter(c: char) -> bool {
    // 是否为大写字母或小写字母
    if c.is_uppercase() || c.is_lowercase() {
        return true;
    }
    match GeneralCategory::of(c) {
    // 大写标题字母
    GeneralCategory::TitlecaseLetter
    // 修饰符字母
    | GeneralCategory::ModifierLetter
    // 其他字母，包括小写字母、大写字母以外的字母
    | GeneralCategory::OtherLetter
    // 字母数字，通常是字母与数字的混合
    | GeneralCategory::LetterNumber => true,
    _ => false,
  }
}

// 字符是否可以作为标识符的开头
fn is_id_start(c: char) -> bool {
    if is_unicode_letter(c) {
        return true;
    }
    match c {
        '$' | '_' | '\\' => true,
        _ => false,
    }
}

// 字符是否可以作为标识符的一部分
fn is_id_part(c: char) -> bool {
    if is_id_start(c) {
        return true;
    }
    let cc = c as u32;
    // Zero Width Non-Joiner and Zero Width Joiner
    if cc == 0x200c || cc == 0x200d {
        return true;
    }
    match GeneralCategory::of(c) {
    // 非间隔标记，包括一些变音符号等
    GeneralCategory::NonspacingMark
    // 间隔标记，包括一些空格修饰符等
    | GeneralCategory::SpacingMark
    // 十进制数字，包括数字 0 到 9
    | GeneralCategory::DecimalNumber
    // 连接标点，包括下划线等
    | GeneralCategory::ConnectorPunctuation => true,
    _ => false,
  }
}

// 是否是单字符转义字符
fn is_single_escape_ch(c: char) -> bool {
    match c {
        '\'' | '"' | '\\' | 'b' | 'f' | 'n' | 'r' | 't' | 'v' => true,
        _ => false,
    }
}

// 将转义字符转换为实际字符
fn escape_ch(c: char) -> char {
    match c {
        '\'' => '\'',
        '"' => '"',
        '\\' => '\\',
        'b' => '\x08',
        'f' => '\x0c',
        'n' => '\x0a',
        'r' => '\x0d',
        't' => '\x09',
        'v' => '\x0b',
        _ => panic!(),
    }
}

// 字符是否不是单字符转义字符、行终止符、ASCII 数字、x、u
fn is_non_escape_ch(c: char) -> bool {
    !is_single_escape_ch(c) && !is_line_terminator(c) && !c.is_ascii_digit() && c != 'x' && c != 'u'
}

pub struct TokenNextNewline {
    tok: Rc<Token>,
    next_is_line_terminator: bool,
}

pub struct Lexer<'a> {
    src: Source<'a>,
    tok: Rc<Token>,
    pub next_is_line_terminator: bool,
    peeked: VecDeque<TokenNextNewline>,
}

#[derive(Debug)]
pub struct LexError {
    pub msg: String,
}

impl LexError {
    fn new(msg: String) -> Self {
        LexError { msg }
    }

    pub fn default() -> Self {
        LexError {
            msg: "".to_string(),
        }
    }
}

impl<'a> Lexer<'a> {
    pub fn new(src: Source<'a>) -> Self {
        Lexer {
            src,
            tok: Rc::new(Token::Nil),
            next_is_line_terminator: false,
            peeked: VecDeque::new(),
        }
    }

    fn next_(&mut self) -> Result<Token, LexError> {
        self.skip_whitespace();
        // 解析标识符、数字、字符串还是符号
        if self.ahead_is_id_start() {
            self.read_name()
        } else if self.ahead_is_decimal_int() {
            self.read_numeric()
        } else if self.ahead_is_string_start() {
            let t = self.src.read().unwrap();
            self.read_string(t)
        } else if !self.ahead_is_eof() {
            self.read_symbol()
        } else {
            Ok(Token::Eof(EofData {
                loc: self.loc().clone(),
            }))
        }
    }

    // 查看下一个token，但不会将其从输入中移除
    pub fn peek(&mut self) -> Result<Rc<Token>, LexError> {
        match self.peeked.front() {
            // 如果 peeked 中有已经查看的token，它会返回该token
            Some(tok) => Ok(tok.tok.clone()),
            // 调用 next_ 方法来获取下一个token，将其存储在 peeked 中，然后返回
            _ => match self.next_() {
                Ok(tok) => {
                    let tok = Rc::new(tok);
                    let next_is_line_terminator = self.ahead_is_line_terminator_or_eof();
                    self.peeked.push_back(TokenNextNewline {
                        tok: tok.clone(),
                        next_is_line_terminator,
                    });
                    Ok(tok)
                }
                Err(e) => Err(e),
            },
        }
    }

    // 获取下一个token，并将其从输入中移除
    pub fn next(&mut self) -> Result<Rc<Token>, LexError> {
        // 检查 peeked 中是否有已经查看的token
        match self.peeked.pop_front() {
            Some(tn) => {
                self.tok = tn.tok;
                self.next_is_line_terminator = tn.next_is_line_terminator;
                Ok(self.tok.clone())
            }
            // 调用 next_ 方法来获取下一个token，并存储在 peeked 中以备后续使用
            _ => match self.next_() {
                Ok(tok) => {
                    self.tok = Rc::new(tok);
                    self.next_is_line_terminator = self.ahead_is_line_terminator_or_eof();
                    Ok(self.tok.clone())
                }
                Err(e) => Err(e),
            },
        }
    }

    // 获取下一个token，但不会将其从输入中移除
    pub fn advance(&mut self) {
        match self.next() {
            Ok(_) => (),
            Err(e) => panic!("{}", e.msg),
        }
    }

    // 解析 Unicode 转义序列
    fn read_unicode_escape_seq(&mut self) -> Option<char> {
        let mut hex = [0, 0, 0, 0];
        for i in 0..hex.len() {
            match self.src.read() {
                Some(c) => {
                    if c.is_ascii_hexdigit() {
                        hex[i] = c as u8;
                    } else {
                        return None;
                    }
                }
                _ => return None,
            }
        }
        let hex = str::from_utf8(&hex).unwrap();
        match u32::from_str_radix(hex, 16) {
            Ok(i) => match char::from_u32(i) {
                Some(c) => Some(c),
                _ => None, // deformed unicode
            },
            _ => None, // deformed hex digits
        }
    }

    // 检查下一个字符是否是标识符的起始字符
    fn ahead_is_id_start(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => is_id_start(c),
            _ => false,
        }
    }
    // 检查下一个字符是否是起始标识符的部分字符
    fn ahead_is_id_part(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => is_id_part(c),
            _ => false,
        }
    }

    fn errmsg(&self) -> String {
        format!(
            "Unexpected char at line: {} column: {}",
            self.src.line, self.src.column
        )
    }

    pub fn pos(&self) -> Position {
        Position {
            line: self.src.line,
            column: self.src.column,
        }
    }

    pub fn loc(&self) -> SourceLoc {
        SourceLoc {
            start: self.pos(),
            end: Position::new(),
        }
    }

    fn fin_loc(&self, loc: SourceLoc) -> SourceLoc {
        let mut loc = loc;
        loc.end = self.pos();
        loc
    }

    fn read_escape_unicode(&mut self, bs: char) -> Result<char, LexError> {
        // Unicode 转义序列以 \u 开头，后跟 4 个十六进制数字
        if bs == '\\' && self.src.test_ahead('u') {
            self.src.advance();
            match self.read_unicode_escape_seq() {
                Some(ec) => Ok(ec),
                _ => Err(LexError::new(self.errmsg())),
            }
        } else {
            Ok(bs)
        }
    }

    fn read_id_part(&mut self) -> Result<String, LexError> {
        let mut val = vec![];
        loop {
            if self.ahead_is_id_part() {
                let c = self.src.read().unwrap();
                match self.read_escape_unicode(c) {
                    Ok(cc) => val.push(cc),
                    Err(e) => return Err(e),
                }
            } else {
                break;
            }
        }
        Ok(val.into_iter().collect())
    }

    // 解析keyword bool null和标识符
    pub fn read_name(&mut self) -> Result<Token, LexError> {
        let loc = self.loc();
        let mut c = self.src.read().unwrap();
        match self.read_escape_unicode(c) {
            Ok(cc) => c = cc,
            Err(e) => return Err(e),
        }
        let mut val = vec![c];
        match self.read_id_part() {
            Ok(cc) => val.extend(cc.chars()),
            Err(e) => return Err(e),
        }
        let val: String = val.into_iter().collect();
        if is_keyword(&val) {
            Ok(Token::Keyword(KeywordData {
                kind: name_to_keyword(&val),
                loc: self.fin_loc(loc),
            }))
        } else if is_ctx_keyword(&val) {
            Ok(Token::ContextualKeyword(CtxKeywordData {
                kind: name_to_ctx_keyword(&val),
                loc: self.fin_loc(loc),
            }))
        } else if is_bool(&val) {
            Ok(Token::BooleanLiteral(BooleanLiteralData {
                kind: name_to_bool(&val),
                loc: self.fin_loc(loc),
            }))
        } else if is_null(&val) {
            Ok(Token::NullLiteral(NullLiteralData {
                loc: self.fin_loc(loc),
            }))
        } else {
            Ok(Token::Identifier(IdentifierData {
                value: val,
                loc: self.fin_loc(loc),
            }))
        }
    }

    // 解析数字
    fn read_decimal_digits(&mut self) -> String {
        let mut ret = String::new();
        loop {
            if let Some(c) = self.src.peek() {
                if c.is_ascii_digit() {
                    ret.push(self.src.read().unwrap());
                    continue;
                }
            }
            break;
        }
        ret
    }

    // 科学计算
    fn read_exponent(&mut self) -> Result<String, LexError> {
        let mut ret = String::new();
        // 使用 e|E
        ret.push(self.src.read().unwrap());
        if let Some(c) = self.src.peek() {
            if c == '+' || c == '-' {
                ret.push(self.src.read().unwrap());
            }
            let digits = self.read_decimal_digits();
            if digits.is_empty() {
                return Err(LexError::new(self.errmsg()));
            } else {
                ret.push_str(digits.as_str());
            }
        }
        Ok(ret)
    }

    fn read_decimal_int_part(&mut self) -> String {
        let mut ret = String::new();
        let c = self.src.read().unwrap();
        ret.push(c);
        if c == '0' {
            return ret;
        }
        ret.push_str(self.read_decimal_digits().as_str());
        ret
    }

    // 解析字符串
    fn read_decimal(&mut self) -> Result<String, LexError> {
        let c = self.src.peek().unwrap();
        let mut ret = String::new();
        let digits_opt = c != '.';
        if c.is_ascii_digit() {
            ret.push_str(self.read_decimal_int_part().as_str());
        }
        // 处理小数部分：这部分代码涉及解析浮点数，即具有小数部分的数字。根据 JavaScript 语法，浮点数的小数部分是可选的。
        // 如果浮点数以小数点 . 开头，那么下一个字符（或字符序列）必须是小数部分。这意味着小数点后面必须紧跟着数字。
        if self.src.test_ahead('.') {
            ret.push(self.src.read().unwrap());
            let digits = self.read_decimal_digits();
            if digits.is_empty() && !digits_opt {
                return Err(LexError::new(self.errmsg()));
            }
            ret.push_str(digits.as_str());
        }
        //如果浮点数以非零数字开头，那么小数部分中的数字是可选的。这意味着可以有整数值而无小数部分。
        if self.src.test_ahead_or('e', 'E') {
            match self.read_exponent() {
                Ok(s) => ret.push_str(s.as_str()),
                err @ Err(_) => return err,
            }
        }

        Ok(ret)
    }

    // 解析十六进制
    fn read_hex(&mut self) -> Result<String, LexError> {
        let mut ret = String::new();
        ret.push(self.src.read().unwrap());
        ret.push(self.src.read().unwrap());
        let mut digits = vec![];
        loop {
            match self.src.peek() {
                Some(c) => {
                    if c.is_ascii_hexdigit() {
                        digits.push(self.src.read().unwrap());
                    } else {
                        break;
                    }
                }
                _ => break,
            }
        }
        if digits.len() == 0 {
            Err(LexError::new(self.errmsg()))
        } else {
            let digits: String = digits.iter().collect();
            ret.push_str(digits.as_str());
            Ok(ret)
        }
    }

    fn ahead_is_decimal_int(&mut self) -> bool {
        if let Some(c) = self.src.peek() {
            if c == '.' {
                match self.src.chs.next() {
                    Some(cc) => {
                        self.src.peeked.push_back(cc);
                        cc.is_ascii_digit()
                    }
                    _ => false,
                }
            } else {
                c.is_ascii_digit()
            }
        } else {
            false
        }
    }

    // 解析数字
    pub fn read_numeric(&mut self) -> Result<Token, LexError> {
        let loc = self.loc();
        let value: Result<String, LexError>;
        let mut is_hex = false;
        if self.src.test_ahead('0') {
            if let Some(c) = self.src.chs.next() {
                self.src.peeked.push_back(c);
                if c == 'x' || c == 'X' {
                    is_hex = true;
                }
            }
        }
        if is_hex {
            value = self.read_hex();
        } else {
            value = self.read_decimal();
        }
        match value {
            Ok(v) => Ok(Token::NumericLiteral(NumericLiteralData {
                value: v,
                loc: self.fin_loc(loc),
            })),
            Err(e) => Err(e),
        }
    }

    fn ahead_is_string_start(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => c == '\'' || c == '"',
            _ => false,
        }
    }

    // 解析转译
    fn read_string_escape_seq(&mut self) -> Result<Option<char>, LexError> {
        self.src.advance(); // 反斜杠
        match self.src.read() {
            Some(mut c) => {
                // 单字符转义字符
                if is_single_escape_ch(c) {
                    c = escape_ch(c);
                    // \0：表示空字符 (NUL)
                } else if c == '0' {
                    c = '\0';
                    // 0 [lookahead ∉ DecimalDigit]
                    if let Some(c) = self.src.peek() {
                        if c.is_ascii_digit() {
                            return Err(LexError::new(self.errmsg()));
                        }
                    }
                    // \x：表示十六进制转义
                } else if c == 'x' {
                    let mut hex = [0, 0];
                    for i in 0..hex.len() {
                        if let Some(cc) = self.src.read() {
                            if cc.is_ascii_hexdigit() {
                                hex[i] = cc as u8;
                                continue;
                            }
                        }
                        return Err(LexError::new(self.errmsg()));
                    }
                    let hex = str::from_utf8(&hex).unwrap();
                    c = char::from_u32(u32::from_str_radix(hex, 16).ok().unwrap()).unwrap()
                    // \u：表示 Unicode 转义
                } else if c == 'u' {
                    match self.read_unicode_escape_seq() {
                        Some(ec) => c = ec,
                        _ => return Err(LexError::new(self.errmsg())),
                    }
                    // 换行符
                } else if is_line_terminator(c) {
                    if c == '\r' && self.src.test_ahead('\n') {
                        self.src.advance();
                        return Err(LexError::new(self.errmsg()));
                    }
                    return Ok(None);
                } else if is_non_escape_ch(c) {
                    // todo
                } else {
                    return Err(LexError::new(self.errmsg()));
                }
                Ok(Some(c))
            }
            _ => Err(LexError::new(self.errmsg())),
        }
    }

    // 解析字符串
    fn read_string(&mut self, t: char) -> Result<Token, LexError> {
        let loc = self.loc();
        let mut ret = String::new();
        loop {
            match self.src.peek() {
                Some(c) => {
                    if c == t {
                        self.src.advance();
                        break;
                    } else if c == '\\' {
                        match self.read_string_escape_seq() {
                            Ok(Some(c)) => ret.push(c),
                            Err(e) => return Err(e),
                            _ => (),
                        }
                    } else {
                        ret.push(self.src.read().unwrap());
                    }
                }
                _ => break,
            }
        }
        Ok(Token::StringLiteral(StringLiteralData {
            value: ret,
            loc: self.fin_loc(loc),
        }))
    }

    fn ahead_is_regexp_start(&mut self) -> bool {
        match self.src.peek() {
            Some('/') => self.tok.is_before_expr(),
            _ => false,
        }
    }

    fn ahead_is_regexp_backslash_seq(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => c == '\\',
            _ => false,
        }
    }

    fn read_regexp_backslash_seq(&mut self) -> Result<String, LexError> {
        let mut ret = vec![self.src.read().unwrap()];
        if self.ahead_is_line_terminator_or_eof() {
            Err(LexError::new(self.errmsg()))
        } else {
            ret.push(self.src.read().unwrap());
            Ok(ret.into_iter().collect())
        }
    }

    fn ahead_is_regexp_class(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => c == '[',
            _ => false,
        }
    }

    fn read_regexp_class(&mut self) -> Result<String, LexError> {
        let mut ret = vec![self.src.read().unwrap()];
        loop {
            if self.ahead_is_regexp_backslash_seq() {
                match self.read_regexp_backslash_seq() {
                    Ok(s) => ret.extend(s.chars()),
                    Err(e) => return Err(e),
                }
            } else if self.ahead_is_line_terminator_or_eof() {
                return Err(LexError::new(self.errmsg()));
            } else {
                let c = self.src.read().unwrap();
                ret.push(c);
                if c == ']' {
                    break;
                }
            };
        }
        Ok(ret.into_iter().collect())
    }

    fn read_regexp_body(&mut self) -> Result<String, LexError> {
        let mut ret = vec![self.src.read().unwrap()];
        loop {
            if self.ahead_is_regexp_backslash_seq() {
                match self.read_regexp_backslash_seq() {
                    Ok(s) => ret.extend(s.chars()),
                    Err(e) => return Err(e),
                }
            } else if self.ahead_is_regexp_class() {
                match self.read_regexp_class() {
                    Ok(s) => ret.extend(s.chars()),
                    Err(e) => return Err(e),
                }
            } else if self.ahead_is_line_terminator_or_eof() {
                return Err(LexError::new(self.errmsg()));
            } else {
                let c = self.src.read().unwrap();
                ret.push(c);
                if c == '/' {
                    break;
                }
            }
        }
        Ok(ret.into_iter().collect())
    }

    fn read_regexp_flags(&mut self) -> Result<String, LexError> {
        let mut ret = vec![];
        loop {
            if self.ahead_is_id_part() {
                match self.read_id_part() {
                    Ok(s) => ret.extend(s.chars()),
                    Err(e) => return Err(e),
                }
            } else {
                break;
            }
        }
        Ok(ret.into_iter().collect())
    }

    // 解析正则
    fn read_regexp(&mut self) -> Result<Token, LexError> {
        let loc = self.loc();
        match self.read_regexp_body() {
            Ok(mut body) => match self.read_regexp_flags() {
                Ok(flags) => {
                    body.push_str(flags.as_str());
                    Ok(Token::RegExpLiteral(RegExpLiteralData {
                        value: body,
                        loc: self.fin_loc(loc),
                    }))
                }
                Err(e) => Err(e),
            },
            Err(e) => Err(e),
        }
    }

    // 解析符号
    fn read_symbol(&mut self) -> Result<Token, LexError> {
        if self.ahead_is_regexp_start() {
            return self.read_regexp();
        }
        let loc = self.loc();
        let mut s = vec![];
        loop {
            if self.ahead_is_whitespace_or_eof() {
                break;
            }
            let c = self.src.peek().unwrap();
            match c {
                '{' | '}' | '(' | ')' | '[' | ']' | '.' | ';' | ',' | '?' | ':' => {
                    s.push(self.src.read().unwrap());
                    break;
                }
                '<' => {
                    // < <<= << <=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead2('<', '=') {
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                    } else if self.src.test_ahead_or('<', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '>' => {
                    // > >>>= >>= >> >=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead3('>', '>', '=') {
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                    } else if self.src.test_ahead2('>', '=') {
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                    } else if self.src.test_ahead_or('>', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '=' => {
                    // = === ==
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead2('=', '=') {
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                    } else if self.src.test_ahead('=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '!' => {
                    // ! != !==
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead2('=', '=') {
                        s.push(self.src.read().unwrap());
                        s.push(self.src.read().unwrap());
                    } else if self.src.test_ahead('=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '+' => {
                    // + ++ +=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead_or('+', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '-' => {
                    // - -- -=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead_or('-', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '&' => {
                    // & && &=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead_or('&', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '|' => {
                    // | || |=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead_or('|', '=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                '*' | '/' | '%' | '^' | '~' => {
                    // pattern pattern=
                    s.push(self.src.read().unwrap());
                    if self.src.test_ahead('=') {
                        s.push(self.src.read().unwrap());
                    }
                    break;
                }
                _ => return Err(LexError::new(self.errmsg())),
            }
        }
        let s: String = s.into_iter().collect();
        if is_symbol(&s) {
            Ok(Token::Symbol(SymbolData {
                kind: name_to_symbol(&s),
                loc: self.fin_loc(loc),
            }))
        } else {
            Err(LexError::new(self.errmsg()))
        }
    }

    // 跳过注释
    fn skip_comment_single(&mut self) {
        self.src.advance2();
        loop {
            match self.src.read() {
                Some(EOL) | None => break,
                _ => (),
            };
        }
    }

    fn skip_comment_multi(&mut self) {
        self.src.advance2();
        loop {
            match self.src.read() {
                Some('*') => {
                    if self.src.test_ahead('/') {
                        self.src.advance();
                        break;
                    }
                }
                None => break,
                _ => (),
            };
        }
    }

    pub fn ahead_is_line_terminator_or_eof(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => is_line_terminator(c),
            _ => true,
        }
    }

    pub fn ahead_is_eof(&mut self) -> bool {
        match self.src.peek() {
            Some(_) => false,
            _ => true,
        }
    }

    pub fn ahead_is_whitespace(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => is_whitespace(c),
            _ => false,
        }
    }

    pub fn ahead_is_whitespace_or_line_terminator(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => c.is_whitespace(),
            _ => false,
        }
    }

    pub fn ahead_is_whitespace_or_eof(&mut self) -> bool {
        match self.src.peek() {
            Some(c) => is_whitespace(c),
            _ => true,
        }
    }

    // 跳过空白字符
    pub fn skip_whitespace(&mut self) {
        loop {
            if self.ahead_is_whitespace_or_line_terminator() {
                self.src.read();
            } else if self.src.test_ahead2('/', '/') {
                self.skip_comment_single();
            } else if self.src.test_ahead2('/', '*') {
                self.skip_comment_multi();
            } else {
                break;
            }
        }
    }
}

```

# 语法解析

## AST

语法分析的最终产物就是 ast，因此我们需要先设计出我的 ast，然后才能进一步设计我们的 parser。

我们的 ast 将会包含 Prog，Stmt，Expr 以及字面量。首先我们来设计字面量，即用于描述不同数据类型的值：

```rust
// 字面量，用于描述不同数据类型的值
#[derive(Debug)]
pub struct RegExpData {
    pub loc: SourceLoc,
    pub value: String,
}

impl RegExpData {
    pub fn new(loc: SourceLoc, value: String) -> Self {
        RegExpData { loc, value }
    }
}

#[derive(Debug)]
pub struct NullData {
    pub loc: SourceLoc,
}

impl NullData {
    pub fn new(loc: SourceLoc) -> Self {
        NullData { loc }
    }
}

#[derive(Debug)]
pub struct UndefData {
    pub loc: SourceLoc,
}

impl UndefData {
    pub fn new(loc: SourceLoc) -> Self {
        UndefData { loc }
    }
}

#[derive(Debug)]
pub struct StringData {
    pub loc: SourceLoc,
    pub value: String,
}

impl StringData {
    pub fn new(loc: SourceLoc, value: String) -> Self {
        StringData { loc, value }
    }
}

#[derive(Debug)]
pub struct BoolData {
    pub loc: SourceLoc,
    pub value: bool,
}

impl BoolData {
    pub fn new(loc: SourceLoc, value: bool) -> Self {
        BoolData { loc, value }
    }
}

#[derive(Debug)]
pub struct NumericData {
    pub loc: SourceLoc,
    pub value: String,
}

impl NumericData {
    pub fn new(loc: SourceLoc, value: String) -> Self {
        NumericData { loc, value }
    }
}

#[derive(Debug)]
pub enum Literal {
    RegExp(RegExpData),
    Null(NullData),
    Undef(UndefData),
    String(StringData),
    Bool(BoolData),
    Numeric(NumericData),
}

impl Literal {
    pub fn is_regexp(&self) -> bool {
        match self {
            Literal::RegExp(_) => true,
            _ => false,
        }
    }

    pub fn is_null(&self) -> bool {
        match self {
            Literal::Null(_) => true,
            _ => false,
        }
    }

    pub fn is_str(&self) -> bool {
        match self {
            Literal::String(_) => true,
            _ => false,
        }
    }

    pub fn is_bool(&self) -> bool {
        match self {
            Literal::Bool(_) => true,
            _ => false,
        }
    }

    pub fn is_num(&self) -> bool {
        match self {
            Literal::Numeric(_) => true,
            _ => false,
        }
    }

    pub fn regexp(&self) -> &RegExpData {
        match self {
            Literal::RegExp(d) => d,
            _ => panic!(),
        }
    }

    pub fn null(&self) -> &NullData {
        match self {
            Literal::Null(d) => d,
            _ => panic!(),
        }
    }

    pub fn str(&self) -> &StringData {
        match self {
            Literal::String(d) => d,
            _ => panic!(),
        }
    }

    pub fn bool(&self) -> &BoolData {
        match self {
            Literal::Bool(d) => d,
            _ => panic!(),
        }
    }

    pub fn num(&self) -> &NumericData {
        match self {
            Literal::Numeric(d) => d,
            _ => panic!(),
        }
    }

    pub fn loc(&self) -> &SourceLoc {
        match self {
            Literal::RegExp(d) => &d.loc,
            Literal::Null(d) => &d.loc,
            Literal::Undef(d) => &d.loc,
            Literal::Numeric(d) => &d.loc,
            Literal::String(d) => &d.loc,
            Literal::Bool(d) => &d.loc,
        }
    }
}

impl From<RegExpData> for Literal {
    fn from(f: RegExpData) -> Self {
        Literal::RegExp(f)
    }
}

impl From<NullData> for Literal {
    fn from(f: NullData) -> Self {
        Literal::Null(f)
    }
}

impl From<UndefData> for Literal {
    fn from(f: UndefData) -> Self {
        Literal::Undef(f)
    }
}

impl From<StringData> for Literal {
    fn from(f: StringData) -> Self {
        Literal::String(f)
    }
}

impl From<BoolData> for Literal {
    fn from(f: BoolData) -> Self {
        Literal::Bool(f)
    }
}

impl From<NumericData> for Literal {
    fn from(f: NumericData) -> Self {
        Literal::Numeric(f)
    }
}
```

下一步实现 Expr，即表达式，比如赋值，创建变量等行为：

```rust

// 表达式
#[derive(Debug)]
pub struct ThisExprData {
    pub loc: SourceLoc,
}

impl ThisExprData {
    pub fn new(loc: SourceLoc) -> Self {
        ThisExprData { loc }
    }
}

#[derive(Debug)]
pub struct IdData {
    pub loc: SourceLoc,
    pub name: String,
}

impl IdData {
    pub fn new(loc: SourceLoc, name: String) -> Self {
        IdData { loc, name }
    }
}

#[derive(Debug)]
pub struct ArrayData {
    pub loc: SourceLoc,
    pub value: Vec<Expr>,
}

#[derive(Debug)]
pub struct ObjectProperty {
    pub loc: SourceLoc,
    pub key: Expr,
    pub value: Expr,
}

#[derive(Debug)]
pub struct ObjectData {
    pub loc: SourceLoc,
    pub properties: Vec<ObjectProperty>,
}

#[derive(Debug)]
pub struct ParenData {
    pub loc: SourceLoc,
    pub value: Expr,
}

#[derive(Debug)]
pub struct FnDec {
    pub loc: SourceLoc,
    pub id: Option<PrimaryExpr>,
    pub params: Vec<PrimaryExpr>,
    pub body: Stmt,
}
// 基本表达式
#[derive(Debug)]
pub enum PrimaryExpr {
    // this.fn = b
    This(ThisExprData),
    Identifier(IdData),
    Literal(Literal),
    // [1,2,3]
    ArrayLiteral(ArrayData),
    ObjectLiteral(ObjectData),
    Parenthesized(ParenData),
    Function(Rc<FnDec>),
}

impl PrimaryExpr {
    pub fn is_id(&self) -> bool {
        match self {
            PrimaryExpr::Identifier(_) => true,
            _ => false,
        }
    }

    pub fn is_this(&self) -> bool {
        match self {
            PrimaryExpr::This(_) => true,
            _ => false,
        }
    }

    pub fn is_literal(&self) -> bool {
        match self {
            PrimaryExpr::Literal(_) => true,
            _ => false,
        }
    }

    pub fn is_array(&self) -> bool {
        match self {
            PrimaryExpr::ArrayLiteral(_) => true,
            _ => false,
        }
    }

    pub fn is_object(&self) -> bool {
        match self {
            PrimaryExpr::ObjectLiteral(_) => true,
            _ => false,
        }
    }

    pub fn is_paren(&self) -> bool {
        match self {
            PrimaryExpr::Parenthesized(_) => true,
            _ => false,
        }
    }

    pub fn is_fn(&self) -> bool {
        match self {
            PrimaryExpr::Function(_) => true,
            _ => false,
        }
    }

    pub fn this(&self) -> &ThisExprData {
        match self {
            PrimaryExpr::This(d) => d,
            _ => panic!(),
        }
    }

    pub fn literal(&self) -> &Literal {
        match self {
            PrimaryExpr::Literal(d) => d,
            _ => panic!(),
        }
    }

    pub fn array(&self) -> &ArrayData {
        match self {
            PrimaryExpr::ArrayLiteral(d) => d,
            _ => panic!(),
        }
    }

    pub fn object(&self) -> &ObjectData {
        match self {
            PrimaryExpr::ObjectLiteral(d) => d,
            _ => panic!(),
        }
    }

    pub fn paren(&self) -> &ParenData {
        match self {
            PrimaryExpr::Parenthesized(d) => d,
            _ => panic!(),
        }
    }

    pub fn id(&self) -> &IdData {
        match self {
            PrimaryExpr::Identifier(d) => d,
            _ => panic!(),
        }
    }

    pub fn fn_expr(&self) -> &Rc<FnDec> {
        match self {
            PrimaryExpr::Function(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn loc(&self) -> &SourceLoc {
        match self {
            PrimaryExpr::This(d) => &d.loc,
            PrimaryExpr::Identifier(d) => &d.loc,
            PrimaryExpr::Literal(d) => &d.loc(),
            PrimaryExpr::ArrayLiteral(d) => &d.loc,
            PrimaryExpr::ObjectLiteral(d) => &d.loc,
            PrimaryExpr::Parenthesized(d) => &d.loc,
            PrimaryExpr::Function(d) => &d.loc,
        }
    }
}

impl From<ThisExprData> for PrimaryExpr {
    fn from(f: ThisExprData) -> Self {
        PrimaryExpr::This(f)
    }
}

impl From<IdData> for PrimaryExpr {
    fn from(f: IdData) -> Self {
        PrimaryExpr::Identifier(f)
    }
}

impl From<Literal> for PrimaryExpr {
    fn from(f: Literal) -> Self {
        PrimaryExpr::Literal(f)
    }
}

impl From<StringData> for PrimaryExpr {
    fn from(f: StringData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<NullData> for PrimaryExpr {
    fn from(f: NullData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<UndefData> for PrimaryExpr {
    fn from(f: UndefData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<RegExpData> for PrimaryExpr {
    fn from(f: RegExpData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<BoolData> for PrimaryExpr {
    fn from(f: BoolData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<NumericData> for PrimaryExpr {
    fn from(f: NumericData) -> Self {
        let node: Literal = f.into();
        node.into()
    }
}

impl From<ArrayData> for PrimaryExpr {
    fn from(f: ArrayData) -> Self {
        PrimaryExpr::ArrayLiteral(f)
    }
}

impl From<FnDec> for PrimaryExpr {
    fn from(f: FnDec) -> Self {
        let expr = Rc::new(f);
        PrimaryExpr::Function(expr)
    }
}

impl From<ParenData> for PrimaryExpr {
    fn from(f: ParenData) -> Self {
        PrimaryExpr::Parenthesized(f)
    }
}

#[derive(Debug)]
pub struct UnaryExpr {
    pub loc: SourceLoc,
    pub op: Token,
    pub argument: Expr,
    pub prefix: bool,
}

#[derive(Debug)]
pub struct BinaryExpr {
    pub loc: SourceLoc,
    pub op: Token,
    pub left: Expr,
    pub right: Expr,
}

#[derive(Debug)]
pub struct MemberExpr {
    pub loc: SourceLoc,
    pub object: Expr,
    pub property: Expr,
    pub computed: bool,
}

#[derive(Debug)]
pub struct NewExpr {
    pub loc: SourceLoc,
    pub callee: Expr,
    pub arguments: Vec<Expr>,
}

#[derive(Debug)]
pub struct CallExpr {
    pub loc: SourceLoc,
    pub callee: Expr,
    pub arguments: Vec<Expr>,
}

#[derive(Debug)]
pub struct CondExpr {
    pub loc: SourceLoc,
    pub test: Expr,
    pub cons: Expr,
    pub alt: Expr,
}

#[derive(Debug)]
pub struct AssignExpr {
    pub loc: SourceLoc,
    pub op: Token,
    pub left: Expr,
    pub right: Expr,
}

#[derive(Debug)]
pub struct SeqExpr {
    pub loc: SourceLoc,
    pub exprs: Vec<Expr>,
}

#[derive(Debug, Clone)]
pub enum Expr {
    Primary(Rc<PrimaryExpr>),
    // func
    Member(Rc<MemberExpr>),
    // new
    New(Rc<NewExpr>),
    // func()
    Call(Rc<CallExpr>),
    // "-" | "+" | "!" | "~" | "typeof" | "void" | "delete"
    Unary(Rc<UnaryExpr>),
    // "==" | "!=" | "===" | "!=="  | "<" | "<=" | ">" | ">=" | "<<" | ">>" | ">>>" | "+" | "-" | "*" | "**" | "/" | "%" | "|" | "^" | "&" | "in" | "instanceof"
    Binary(Rc<BinaryExpr>),
    // "=" | "+=" | "-=" | "*=" | "**=" | "/=" | "%=" | "<<=" | ">>=" | ">>>=" | "|=" | "^=" | "&=" | "||=" | "&&=" | "??="
    Assignment(Rc<AssignExpr>),
    // isMember ? '$2.00' : '$10.00';
    Conditional(Rc<CondExpr>),
    Sequence(Rc<SeqExpr>),
}

impl Expr {
    pub fn is_unary(&self) -> bool {
        match self {
            Expr::Unary(_) => true,
            _ => false,
        }
    }

    pub fn is_member(&self) -> bool {
        match self {
            Expr::Member(_) => true,
            _ => false,
        }
    }

    pub fn is_new(&self) -> bool {
        match self {
            Expr::New(_) => true,
            _ => false,
        }
    }

    pub fn is_call(&self) -> bool {
        match self {
            Expr::Call(_) => true,
            _ => false,
        }
    }

    pub fn is_bin(&self) -> bool {
        match self {
            Expr::Binary(_) => true,
            _ => false,
        }
    }

    pub fn is_cond(&self) -> bool {
        match self {
            Expr::Conditional(_) => true,
            _ => false,
        }
    }

    pub fn is_assign(&self) -> bool {
        match self {
            Expr::Assignment(_) => true,
            _ => false,
        }
    }

    pub fn is_seq(&self) -> bool {
        match self {
            Expr::Sequence(_) => true,
            _ => false,
        }
    }

    pub fn primary(&self) -> &PrimaryExpr {
        match self {
            Expr::Primary(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn member(&self) -> &MemberExpr {
        match self {
            Expr::Member(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn unary(&self) -> &UnaryExpr {
        match self {
            Expr::Unary(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn new_expr(&self) -> &NewExpr {
        match self {
            Expr::New(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn call_expr(&self) -> &CallExpr {
        match self {
            Expr::Call(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn bin_expr(&self) -> &BinaryExpr {
        match self {
            Expr::Binary(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn cond_expr(&self) -> &CondExpr {
        match self {
            Expr::Conditional(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn assign_expr(&self) -> &AssignExpr {
        match self {
            Expr::Assignment(expr) => expr,
            _ => panic!(),
        }
    }

    pub fn seq_expr(&self) -> &SeqExpr {
        match self {
            Expr::Sequence(expr) => expr,
            _ => panic!(),
        }
    }
}

impl From<UnaryExpr> for Expr {
    fn from(f: UnaryExpr) -> Self {
        Expr::Unary(Rc::new(f))
    }
}

impl From<PrimaryExpr> for Expr {
    fn from(f: PrimaryExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Primary(expr)
    }
}

impl From<NewExpr> for Expr {
    fn from(f: NewExpr) -> Self {
        let expr = Rc::new(f);
        Expr::New(expr)
    }
}

impl From<CallExpr> for Expr {
    fn from(f: CallExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Call(expr)
    }
}

impl From<BinaryExpr> for Expr {
    fn from(f: BinaryExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Binary(expr)
    }
}

impl From<CondExpr> for Expr {
    fn from(f: CondExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Conditional(expr)
    }
}

impl From<AssignExpr> for Expr {
    fn from(f: AssignExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Assignment(expr)
    }
}

impl From<SeqExpr> for Expr {
    fn from(f: SeqExpr) -> Self {
        let expr = Rc::new(f);
        Expr::Sequence(expr)
    }
}

```

最后我们来实现 Stmt，对语句范畴的定义描述，而 Prog 实际上就是 Stmt 的一个合集：

```rust

// 对语句范畴的定义描述

#[derive(Debug)]
pub struct ExprStmt {
    pub expr: Expr,
}

#[derive(Debug, Clone)]
pub struct BlockStmt {
    pub loc: SourceLoc,
    pub body: Vec<Stmt>,
}

#[derive(Debug)]
pub struct VarDecor {
    pub id: PrimaryExpr,
    pub init: Option<Expr>,
}

#[derive(Debug)]
pub struct VarDec {
    pub loc: SourceLoc,
    pub decs: Vec<VarDecor>,
}

#[derive(Debug)]
pub struct EmptyStmt {
    pub loc: SourceLoc,
}

#[derive(Debug)]
pub struct IfStmt {
    pub loc: SourceLoc,
    pub test: Expr,
    pub cons: Stmt,
    pub alt: Option<Stmt>,
}

#[derive(Debug)]
pub enum ForFirst {
    VarDec(Rc<VarDec>),
    Expr(Expr),
}

#[derive(Debug)]
pub struct ForStmt {
    pub loc: SourceLoc,
    pub init: Option<ForFirst>,
    pub test: Option<Expr>,
    pub update: Option<Expr>,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct ForInStmt {
    pub loc: SourceLoc,
    pub left: ForFirst,
    pub right: Expr,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct DoWhileStmt {
    pub loc: SourceLoc,
    pub test: Expr,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct WhileStmt {
    pub loc: SourceLoc,
    pub test: Expr,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct ContStmt {
    pub loc: SourceLoc,
}

#[derive(Debug)]
pub struct BreakStmt {
    pub loc: SourceLoc,
}

#[derive(Debug)]
pub struct ReturnStmt {
    pub loc: SourceLoc,
    pub argument: Option<Expr>,
}

#[derive(Debug)]
pub struct WithStmt {
    pub loc: SourceLoc,
    pub object: Expr,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct SwitchCase {
    pub test: Option<Expr>,
    pub cons: Vec<Stmt>,
}

#[derive(Debug)]
pub struct SwitchStmt {
    pub loc: SourceLoc,
    pub discrim: Expr,
    pub cases: Vec<SwitchCase>,
}

#[derive(Debug)]
pub struct ThrowStmt {
    pub loc: SourceLoc,
    pub argument: Expr,
}

#[derive(Debug)]
pub struct CatchClause {
    pub id: PrimaryExpr,
    pub body: Stmt,
}

#[derive(Debug)]
pub struct TryStmt {
    pub loc: SourceLoc,
    pub block: Stmt,
    pub handler: Option<CatchClause>,
    pub finalizer: Option<Stmt>,
}

#[derive(Debug)]
pub struct DebugStmt {
    pub loc: SourceLoc,
}

#[derive(Debug, Clone)]
pub enum Stmt {
    Block(Rc<BlockStmt>),
    VarDec(Rc<VarDec>),
    Empty(Rc<EmptyStmt>),
    Expr(Rc<ExprStmt>),
    If(Rc<IfStmt>),
    For(Rc<ForStmt>),
    ForIn(Rc<ForInStmt>),
    DoWhile(Rc<DoWhileStmt>),
    While(Rc<WhileStmt>),
    Cont(Rc<ContStmt>),
    Break(Rc<BreakStmt>),
    Return(Rc<ReturnStmt>),
    With(Rc<WithStmt>),
    Switch(Rc<SwitchStmt>),
    Throw(Rc<ThrowStmt>),
    Try(Rc<TryStmt>),
    Debugger(Rc<DebugStmt>),
    Function(Rc<FnDec>),
}

impl From<BlockStmt> for Stmt {
    fn from(f: BlockStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Block(expr)
    }
}

impl From<ExprStmt> for Stmt {
    fn from(f: ExprStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Expr(expr)
    }
}

impl From<VarDec> for Stmt {
    fn from(f: VarDec) -> Self {
        let expr = Rc::new(f);
        Stmt::VarDec(expr)
    }
}

impl From<IfStmt> for Stmt {
    fn from(f: IfStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::If(expr)
    }
}

impl From<ForInStmt> for Stmt {
    fn from(f: ForInStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::ForIn(expr)
    }
}

impl From<ForStmt> for Stmt {
    fn from(f: ForStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::For(expr)
    }
}

impl From<DoWhileStmt> for Stmt {
    fn from(f: DoWhileStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::DoWhile(expr)
    }
}

impl From<WhileStmt> for Stmt {
    fn from(f: WhileStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::While(expr)
    }
}

impl From<ContStmt> for Stmt {
    fn from(f: ContStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Cont(expr)
    }
}

impl From<BreakStmt> for Stmt {
    fn from(f: BreakStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Break(expr)
    }
}

impl From<ReturnStmt> for Stmt {
    fn from(f: ReturnStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Return(expr)
    }
}

impl From<EmptyStmt> for Stmt {
    fn from(f: EmptyStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Empty(expr)
    }
}

impl From<WithStmt> for Stmt {
    fn from(f: WithStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::With(expr)
    }
}

impl From<SwitchStmt> for Stmt {
    fn from(f: SwitchStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Switch(expr)
    }
}

impl From<DebugStmt> for Stmt {
    fn from(f: DebugStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Debugger(expr)
    }
}

impl From<TryStmt> for Stmt {
    fn from(f: TryStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Try(expr)
    }
}

impl From<ThrowStmt> for Stmt {
    fn from(f: ThrowStmt) -> Self {
        let expr = Rc::new(f);
        Stmt::Throw(expr)
    }
}

impl From<FnDec> for Stmt {
    fn from(f: FnDec) -> Self {
        let expr = Rc::new(f);
        Stmt::Function(expr)
    }
}

impl Stmt {
    pub fn is_block(&self) -> bool {
        match self {
            Stmt::Block(_) => true,
            _ => false,
        }
    }

    pub fn is_expr(&self) -> bool {
        match self {
            Stmt::Expr(_) => true,
            _ => false,
        }
    }

    pub fn is_var(&self) -> bool {
        match self {
            Stmt::VarDec(_) => true,
            _ => false,
        }
    }

    pub fn is_if(&self) -> bool {
        match self {
            Stmt::If(_) => true,
            _ => false,
        }
    }

    pub fn is_for(&self) -> bool {
        match self {
            Stmt::For(_) => true,
            _ => false,
        }
    }

    pub fn is_for_in(&self) -> bool {
        match self {
            Stmt::ForIn(_) => true,
            _ => false,
        }
    }

    pub fn is_do_while(&self) -> bool {
        match self {
            Stmt::DoWhile(_) => true,
            _ => false,
        }
    }

    pub fn is_while_stmt(&self) -> bool {
        match self {
            Stmt::While(_) => true,
            _ => false,
        }
    }

    pub fn is_ret(&self) -> bool {
        match self {
            Stmt::Return(_) => true,
            _ => false,
        }
    }

    pub fn is_cont(&self) -> bool {
        match self {
            Stmt::Cont(_) => true,
            _ => false,
        }
    }

    pub fn is_break(&self) -> bool {
        match self {
            Stmt::Break(_) => true,
            _ => false,
        }
    }

    pub fn is_empty(&self) -> bool {
        match self {
            Stmt::Empty(_) => true,
            _ => false,
        }
    }

    pub fn is_with(&self) -> bool {
        match self {
            Stmt::With(_) => true,
            _ => false,
        }
    }

    pub fn is_switch(&self) -> bool {
        match self {
            Stmt::Switch(_) => true,
            _ => false,
        }
    }

    pub fn is_debug(&self) -> bool {
        match self {
            Stmt::Debugger(_) => true,
            _ => false,
        }
    }

    pub fn is_try(&self) -> bool {
        match self {
            Stmt::Try(_) => true,
            _ => false,
        }
    }

    pub fn is_throw(&self) -> bool {
        match self {
            Stmt::Throw(_) => true,
            _ => false,
        }
    }

    pub fn is_fn(&self) -> bool {
        match self {
            Stmt::Function(_) => true,
            _ => false,
        }
    }

    pub fn block(&self) -> &BlockStmt {
        match self {
            Stmt::Block(s) => s,
            _ => panic!(),
        }
    }

    pub fn expr(&self) -> &ExprStmt {
        match self {
            Stmt::Expr(s) => s,
            _ => panic!(),
        }
    }

    pub fn var_dec(&self) -> &VarDec {
        match self {
            Stmt::VarDec(s) => s,
            _ => panic!(),
        }
    }

    pub fn if_stmt(&self) -> &IfStmt {
        match self {
            Stmt::If(s) => s,
            _ => panic!(),
        }
    }

    pub fn for_stmt(&self) -> &ForStmt {
        match self {
            Stmt::For(s) => s,
            _ => panic!(),
        }
    }

    pub fn for_in(&self) -> &ForInStmt {
        match self {
            Stmt::ForIn(s) => s,
            _ => panic!(),
        }
    }

    pub fn do_while(&self) -> &DoWhileStmt {
        match self {
            Stmt::DoWhile(s) => s,
            _ => panic!(),
        }
    }

    pub fn while_stmt(&self) -> &WhileStmt {
        match self {
            Stmt::While(s) => s,
            _ => panic!(),
        }
    }

    pub fn cont(&self) -> &ContStmt {
        match self {
            Stmt::Cont(s) => s,
            _ => panic!(),
        }
    }

    pub fn break_stmt(&self) -> &BreakStmt {
        match self {
            Stmt::Break(s) => s,
            _ => panic!(),
        }
    }

    pub fn ret_stmt(&self) -> &ReturnStmt {
        match self {
            Stmt::Return(s) => s,
            _ => panic!(),
        }
    }

    pub fn empty(&self) -> &EmptyStmt {
        match self {
            Stmt::Empty(s) => s,
            _ => panic!(),
        }
    }

    pub fn with_stmt(&self) -> &WithStmt {
        match self {
            Stmt::With(s) => s,
            _ => panic!(),
        }
    }

    pub fn switch_stmt(&self) -> &SwitchStmt {
        match self {
            Stmt::Switch(s) => s,
            _ => panic!(),
        }
    }

    pub fn debug_stmt(&self) -> &DebugStmt {
        match self {
            Stmt::Debugger(s) => s,
            _ => panic!(),
        }
    }

    pub fn try_stmt(&self) -> &TryStmt {
        match self {
            Stmt::Try(s) => s,
            _ => panic!(),
        }
    }

    pub fn throw_stmt(&self) -> &ThrowStmt {
        match self {
            Stmt::Throw(s) => s,
            _ => panic!(),
        }
    }

    pub fn fn_dec(&self) -> &FnDec {
        match self {
            Stmt::Function(s) => s,
            _ => panic!(),
        }
    }
}

#[derive(Debug)]
pub struct Prog {
    pub body: Vec<Stmt>,
}

```

## parser

当我们定义完我们的 ast 后，我们下一步就是现实我们的 parser 了，在这里，我使用了递归下降作为语法解析，简单来说就是核心的两个公式：

- S = aS | bS'
- S' = a | Σ

S 代表的是代码的整体，a 和 b 是终结符，不可再分的 token，解析到这就结束了，Σ 代表的是空。根据这种定义，我们可以一直递归解析下去，直到结果只有 a,b,Σ。

```rust

pub struct Parser<'a> {
    pub lexer: &'a mut Lexer<'a>,
    in_loop_flags: Vec<bool>,
}

impl<'a> Parser<'a> {
    pub fn new(lexer: &'a mut Lexer<'a>) -> Self {
        Parser {
            lexer,
            in_loop_flags: vec![],
        }
    }

    fn enter_loop(&mut self) {
        self.in_loop_flags.push(true);
    }

    fn leave_loop(&mut self) {
        self.in_loop_flags.pop();
    }

    fn in_loop(&self) -> bool {
        match self.in_loop_flags.last() {
            Some(_) => true,
            _ => false,
        }
    }

    pub fn prog(&mut self) -> Result<Prog, ParsingError> {
        let mut body = vec![];
        loop {
            match self.lexer.peek() {
                Ok(tok) => {
                    if tok.is_eof() {
                        break;
                    }
                }
                Err(e) => return Err(e.into()),
            }
            match self.stmt() {
                Ok(stmt) => body.push(stmt),
                Err(e) => return Err(e),
            }
        }
        Ok(Prog { body })
    }

    fn stmt(&mut self) -> Result<Stmt, ParsingError> {
        if self.ahead_is_symbol(Symbol::BraceL) {
            self.block_stmt()
        } else if self.ahead_is_keyword(Keyword::Var) {
            self.var_dec_stmt(false, false)
        } else if self.ahead_is_keyword(Keyword::If) {
            self.if_stmt()
        } else if self.ahead_is_keyword(Keyword::For) {
            self.for_stmt()
        } else if self.ahead_is_keyword(Keyword::Do) {
            self.do_while()
        } else if self.ahead_is_keyword(Keyword::While) {
            self.while_stmt()
        } else if self.ahead_is_keyword(Keyword::Continue) {
            self.cont_stmt()
        } else if self.ahead_is_keyword(Keyword::Break) {
            self.break_stmt()
        } else if self.ahead_is_keyword(Keyword::Return) {
            self.ret_stmt()
        } else if self.ahead_is_keyword(Keyword::With) {
            self.with_stmt()
        } else if self.ahead_is_keyword(Keyword::Switch) {
            self.switch_stmt()
        } else if self.ahead_is_keyword(Keyword::Debugger) {
            self.debug_stmt()
        } else if self.ahead_is_keyword(Keyword::Throw) {
            self.throw_stmt()
        } else if self.ahead_is_keyword(Keyword::Try) {
            self.try_stmt()
        } else if self.ahead_is_keyword(Keyword::Function) {
            self.fn_dec_stmt()
        } else if self.ahead_is_symbol(Symbol::Semi) {
            self.empty_stmt()
        } else {
            self.expr_stmt()
        }
    }

    fn fn_dec_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let tok = self.lexer.next().ok().unwrap();
        let expr = match self.fn_expr(&tok) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        let fun = expr.fn_expr().clone();
        Ok(Stmt::Function(fun))
    }

    fn throw_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();

        let argument = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        loc.end = self.pos();
        Ok(ThrowStmt { loc, argument }.into())
    }

    fn try_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();

        let block = match self.block_stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        let mut handler = None;
        let mut finalizer = None;
        if self.ahead_is_keyword(Keyword::Catch) {
            handler = match self.try_catch_clause() {
                Ok(cc) => Some(cc),
                Err(e) => return Err(e),
            };
        } else if self.ahead_is_keyword(Keyword::Finally) {
            finalizer = match self.block_stmt() {
                Ok(stmt) => Some(stmt),
                Err(e) => return Err(e),
            };
        }

        loc.end = self.pos();
        let stmt = TryStmt {
            loc,
            block,
            handler,
            finalizer,
        };
        Ok(stmt.into())
    }

    fn try_catch_clause(&mut self) -> Result<CatchClause, ParsingError> {
        self.lexer.advance();

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let id = match self.primary_expr() {
            Ok(id) => id,
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let body = match self.block_stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        Ok(CatchClause { id, body }.into())
    }

    fn debug_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();
        loc.end = self.pos();
        Ok(DebugStmt { loc }.into())
    }

    fn switch_stmt(&mut self) -> Result<Stmt, ParsingError> {
        self.enter_loop();
        let mut loc = self.loc();
        self.lexer.advance();

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let discrim = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        match self.ahead_is_symbol(Symbol::BraceL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let mut cases = vec![];
        let mut met_default = false;
        loop {
            if self.ahead_is_symbol(Symbol::BraceR) {
                self.lexer.advance();
                break;
            } else if self.ahead_is_keyword(Keyword::Case) {
                match self.switch_case() {
                    Ok(case) => cases.push(case),
                    Err(e) => return Err(e),
                };
            } else if self.ahead_is_keyword(Keyword::Default) {
                if !met_default {
                    met_default = true;
                    match self.switch_default() {
                        Ok(case) => cases.push(case),
                        Err(e) => return Err(e),
                    };
                } else {
                    return Err(ParserError::at(&self.loc()).into());
                }
            }
        }

        loc.end = self.pos();
        let switch = SwitchStmt {
            loc,
            discrim,
            cases,
        };
        self.leave_loop();
        Ok(switch.into())
    }

    fn switch_default(&mut self) -> Result<SwitchCase, ParsingError> {
        self.lexer.advance();

        match self.ahead_is_symbol(Symbol::Colon) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let mut cons = vec![];
        loop {
            let stmt = match self.stmt() {
                Ok(stmt) => stmt,
                Err(e) => return Err(e),
            };
            cons.push(stmt);
            if self.ahead_is_keyword(Keyword::Case) || self.ahead_is_symbol(Symbol::BraceR) {
                break;
            }
        }

        Ok(SwitchCase { test: None, cons })
    }

    fn switch_case(&mut self) -> Result<SwitchCase, ParsingError> {
        self.lexer.advance();

        let test = match self.expr(false) {
            Ok(expr) => Some(expr),
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::Colon) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let mut cons = vec![];
        loop {
            let stmt = match self.stmt() {
                Ok(stmt) => stmt,
                Err(e) => return Err(e),
            };
            cons.push(stmt);
            if self.ahead_is_keyword(Keyword::Case) || self.ahead_is_symbol(Symbol::BraceR) {
                break;
            }
        }

        Ok(SwitchCase { test, cons })
    }

    fn with_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let object = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let body = match self.stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        loc.end = self.pos();
        let stmt = WithStmt { loc, object, body };
        Ok(stmt.into())
    }

    fn empty_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();
        loc.end = self.pos();
        Ok(EmptyStmt { loc }.into())
    }

    fn ret_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let loc = self.loc();
        self.lexer.advance();

        let mut ret = ReturnStmt {
            loc,
            argument: None,
        };
        if self.lexer.next_is_line_terminator {
            if self.ahead_is_symbol(Symbol::Semi) {
                self.lexer.advance();
            }
            ret.loc.end = self.pos();
            return Ok(ret.into());
        }

        if !self.ahead_is_symbol(Symbol::BraceR) {
            if self.ahead_is_symbol(Symbol::Semi) {
                self.lexer.advance();
            } else {
                ret.argument = match self.expr(false) {
                    Ok(expr) => Some(expr),
                    Err(e) => return Err(e),
                };
            }
        }

        ret.loc.end = self.pos();
        Ok(ret.into())
    }

    fn break_stmt(&mut self) -> Result<Stmt, ParsingError> {
        if !self.in_loop() {
            return Err(ParserError::at(&self.loc()).into());
        }

        let mut loc = self.loc();
        self.lexer.advance();
        loc.end = self.pos();
        if self.ahead_is_symbol(Symbol::Semi) {
            self.lexer.advance();
        }
        Ok(BreakStmt { loc }.into())
    }

    fn cont_stmt(&mut self) -> Result<Stmt, ParsingError> {
        if !self.in_loop() {
            return Err(ParserError::at(&self.loc()).into());
        }

        let mut loc = self.loc();
        self.lexer.advance();
        loc.end = self.pos();
        if self.ahead_is_symbol(Symbol::Semi) {
            self.lexer.advance();
        }
        Ok(ContStmt { loc }.into())
    }

    fn while_stmt(&mut self) -> Result<Stmt, ParsingError> {
        self.enter_loop();
        let mut loc = self.loc();
        self.lexer.advance();

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let test = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let body = match self.stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        loc.end = self.pos();
        let stmt = WhileStmt { loc, test, body };
        self.leave_loop();
        Ok(stmt.into())
    }

    fn do_while(&mut self) -> Result<Stmt, ParsingError> {
        self.enter_loop();
        let mut loc = self.loc();
        self.lexer.advance();

        let body = match self.stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        match self.ahead_is_keyword(Keyword::While) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        let test = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        };

        loc.end = self.pos();
        let stmt = DoWhileStmt { loc, test, body };
        self.leave_loop();
        Ok(stmt.into())
    }

    fn for_stmt(&mut self) -> Result<Stmt, ParsingError> {
        self.enter_loop();
        let mut loc = self.loc();
        self.lexer.advance();

        if !self.ahead_is_symbol(Symbol::ParenL) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let mut first = None;
        let mut left = None;
        if self.ahead_is_keyword(Keyword::Var) {
            first = match self.for_var() {
                Ok(expr) => Some(expr),
                Err(e) => return Err(e),
            };
        } else if !self.ahead_is_symbol(Symbol::Semi) {
            left = match self.expr(true) {
                Ok(expr) => Some(expr),
                Err(e) => return Err(e),
            };
        }

        let is_for_in = self.ahead_is_keyword(Keyword::In);

        let mut for_in_left = None;
        if first.is_some() {
            let first = first.unwrap();
            if is_for_in && first.decs.len() > 1 {
                return Err(ParserError::at(&first.loc).into());
            }
            for_in_left = Some(ForFirst::VarDec(first));
        } else if left.is_some() {
            for_in_left = Some(ForFirst::Expr(left.unwrap()));
        } else if is_for_in {
            return Err(ParserError::at(&self.loc()).into());
        }

        if is_for_in {
            self.lexer.advance();

            let right = match self.expr(false) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            };

            if !self.ahead_is_symbol(Symbol::ParenR) {
                return Err(ParserError::at(&self.loc()).into());
            }
            self.lexer.advance();

            let body = match self.stmt() {
                Ok(stmt) => stmt,
                Err(e) => return Err(e),
            };

            loc.end = self.pos();
            self.leave_loop();
            return Ok(ForInStmt {
                loc,
                left: for_in_left.unwrap(),
                right,
                body,
            }
            .into());
        }

        if !self.ahead_is_symbol(Symbol::Semi) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let test = match self.ahead_is_symbol(Symbol::Semi) {
            true => None,
            false => match self.expr(false) {
                Ok(expr) => Some(expr),
                Err(e) => return Err(e),
            },
        };
        if !self.ahead_is_symbol(Symbol::Semi) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let update = match self.ahead_is_symbol(Symbol::ParenR) {
            true => None,
            false => match self.expr(false) {
                Ok(expr) => Some(expr),
                Err(e) => return Err(e),
            },
        };

        if !self.ahead_is_symbol(Symbol::ParenR) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let body = match self.stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        loc.end = self.pos();
        Ok(ForStmt {
            loc,
            init: for_in_left,
            test,
            update,
            body,
        }
        .into())
    }

    fn for_var(&mut self) -> Result<Rc<VarDec>, ParsingError> {
        match self.var_dec_stmt(true, true) {
            Ok(stmt) => match stmt {
                Stmt::VarDec(stmt) => Ok(stmt.clone()),
                _ => panic!(), // unreachable
            },
            Err(e) => Err(e),
        }
    }

    fn if_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();

        if !self.ahead_is_symbol(Symbol::ParenL) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let test = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        if !self.ahead_is_symbol(Symbol::ParenR) {
            return Err(ParserError::at(&self.loc()).into());
        }
        self.lexer.advance();

        let cons = match self.stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        let mut alt: Option<Stmt> = None;
        if self.ahead_is_keyword(Keyword::Else) {
            self.lexer.advance();
            alt = match self.stmt() {
                Ok(stmt) => Some(stmt),
                Err(e) => return Err(e),
            };
        }

        loc.end = self.pos();
        let s = IfStmt {
            loc,
            test,
            cons,
            alt,
        };
        Ok(s.into())
    }

    fn var_dec_stmt(&mut self, not_in: bool, leave_semi: bool) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();
        let mut decs = vec![];
        loop {
            match self.var_decor(not_in) {
                Ok(dec) => decs.push(dec),
                Err(e) => return Err(e),
            }
            if self.ahead_is_symbol(Symbol::Comma) {
                self.lexer.advance();
                continue;
            }
            if self.ahead_is_symbol(Symbol::Semi) && !leave_semi {
                self.lexer.advance();
            }
            break;
        }
        loc.end = self.pos();
        let var_dec = VarDec { loc, decs };
        Ok(var_dec.into())
    }

    fn var_decor(&mut self, not_in: bool) -> Result<VarDecor, ParsingError> {
        let id = match self.primary_expr() {
            Ok(expr) => {
                if !expr.is_id() {
                    return Err(ParserError::at(expr.literal().loc()).into());
                }
                expr
            }
            Err(e) => return Err(e),
        };
        let mut dec = VarDecor { id, init: None };
        if !self.ahead_is_symbol(Symbol::Assign) {
            return Ok(dec);
        }
        self.lexer.advance();
        dec.init = match self.cond_expr(not_in) {
            Ok(expr) => Some(expr),
            Err(e) => return Err(e),
        };
        return Ok(dec);
    }

    fn block_stmt(&mut self) -> Result<Stmt, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();
        let mut body = vec![];
        loop {
            match self.lexer.peek() {
                Ok(tok) => {
                    if tok.is_symbol_kind(Symbol::BraceR) {
                        self.lexer.advance();
                        break;
                    }
                    match self.stmt() {
                        Ok(stmt) => body.push(stmt),
                        Err(e) => return Err(e),
                    }
                }
                Err(e) => return Err(e.into()),
            }
        }
        loc.end = self.pos();
        let b = BlockStmt { loc, body };
        Ok(b.into())
    }

    fn expr_stmt(&mut self) -> Result<Stmt, ParsingError> {
        match self.expr(false) {
            Ok(expr) => {
                if self.ahead_is_symbol(Symbol::Semi) {
                    self.lexer.advance();
                }
                Ok(ExprStmt { expr }.into())
            }
            Err(e) => Err(e),
        }
    }

    fn expr(&mut self, not_in: bool) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let first = match self.assign_expr(not_in, None) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        let tok = self.lexer.peek();
        if tok.is_err() {
            return Ok(first);
        }
        let tok = tok.ok().unwrap();
        if !tok.is_symbol_kind(Symbol::Comma) {
            return Ok(first);
        }
        self.lexer.advance();

        let mut seq: Vec<Expr> = vec![first];
        loop {
            let expr = match self.assign_expr(not_in, None) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            };
            seq.push(expr);
            if self.ahead_is_symbol(Symbol::Comma) {
                self.lexer.advance();
            } else {
                break;
            }
        }
        loc.end = self.pos();
        let seq = SeqExpr { loc, exprs: seq };
        Ok(seq.into())
    }

    fn ahead_is_symbol_assign(&mut self) -> bool {
        match self.lexer.peek() {
            Err(_) => false,
            Ok(tok) => tok.is_symbol_assign(),
        }
    }

    fn assign_expr(&mut self, not_in: bool, left: Option<Expr>) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let lhs = match left {
            Some(expr) => expr,
            _ => match self.cond_expr(not_in) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            },
        };

        if !self.ahead_is_symbol_assign() {
            return Ok(lhs);
        }
        let op = self.lexer.next().ok().unwrap().deref().clone();

        let mut rhs = match self.cond_expr(not_in) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        if self.ahead_is_symbol_assign() {
            rhs = match self.assign_expr(not_in, Some(rhs)) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            };
        }

        loc.end = self.pos();
        let assign = AssignExpr {
            loc,
            op,
            left: lhs,
            right: rhs,
        };
        Ok(assign.into())
    }

    fn cond_expr(&mut self, not_in: bool) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let test = match self.expr_op(None, 0, not_in) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        let tok = self.lexer.peek();
        if tok.is_err() {
            return Ok(test);
        }
        let tok = tok.ok().unwrap();
        if !tok.is_symbol_kind(Symbol::Conditional) {
            return Ok(test);
        }
        self.lexer.advance();

        let cons = match self.assign_expr(not_in, None) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        match self.lexer.next() {
            Ok(tok) => match tok.is_symbol_kind(Symbol::Colon) {
                false => return Err(ParserError::at_tok(&tok).into()),
                true => (),
            },
            Err(e) => return Err(e.into()),
        }

        let alt = match self.assign_expr(not_in, None) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };

        loc.end = self.pos();
        let cond = CondExpr {
            loc,
            test,
            cons,
            alt,
        };
        Ok(cond.into())
    }

    fn expr_op(
        &mut self,
        lhs: Option<Expr>,
        min_pcd: i32,
        not_in: bool,
    ) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let mut lhs = match lhs {
            Some(expr) => expr,
            _ => match self.unary_expr() {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            },
        };
        loop {
            let ahead = match self.lexer.peek() {
                Ok(tok) => tok,
                Err(_) => break,
            };
            let pcd;
            if ahead.is_symbol_bin() {
                pcd = ahead.symbol_pcd();
            } else if ahead.is_keyword_bin(not_in) {
                pcd = ahead.keyword_pcd()
            } else {
                break;
            }
            if pcd < min_pcd {
                break;
            }
            self.lexer.advance();
            let op = ahead.deref().clone();
            let mut rhs = match self.unary_expr() {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            };
            if let Ok(next_ok) = self.lexer.peek() {
                if next_ok.is_symbol_bin() {
                    rhs = match self.expr_op(Some(rhs), pcd, not_in) {
                        Ok(expr) => expr,
                        Err(e) => return Err(e),
                    };
                }
            }
            loc.end = self.pos();
            let bin = BinaryExpr {
                loc,
                op,
                left: lhs,
                right: rhs,
            };
            lhs = bin.into();
        }
        Ok(lhs)
    }

    fn postfix_expr(&mut self) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        match self.lhs_expr() {
            Ok(expr) => {
                if let Ok(tok) = self.lexer.peek() {
                    if tok.is_symbol_kind(Symbol::Inc) || tok.is_symbol_kind(Symbol::Dec) {
                        let tok = self.lexer.next().ok().unwrap();
                        loc.end = self.pos();
                        let expr: Expr = UnaryExpr {
                            loc,
                            op: tok.deref().clone(),
                            argument: expr,
                            prefix: false,
                        }
                        .into();
                        return Ok(expr);
                    }
                }
                Ok(expr)
            }
            Err(e) => Err(e),
        }
    }

    fn unary_expr(&mut self) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let tok = match self.lexer.peek() {
            Ok(tok) => tok,
            Err(e) => return Err(e.into()),
        };
        let ks = vec![Keyword::Delete, Keyword::Void, Keyword::Typeof];
        let ss = vec![
            Symbol::Inc,
            Symbol::Dec,
            Symbol::Add,
            Symbol::Sub,
            Symbol::BitNot,
            Symbol::Not,
        ];
        if tok.is_keyword_kind_in(&ks) || tok.is_symbol_kind_in(&ss) {
            let op = self.lexer.next().ok().unwrap().deref().clone();
            let argument = match self.unary_expr() {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            };
            loc.end = self.pos();
            let expr = UnaryExpr {
                loc,
                op,
                argument,
                prefix: true,
            };
            Ok(expr.into())
        } else {
            self.postfix_expr()
        }
    }

    fn lhs_expr(&mut self) -> Result<Expr, ParsingError> {
        if self.ahead_is_keyword(Keyword::New) {
            self.new_expr()
        } else {
            self.call_expr(None)
        }
    }

    fn ahead_is_keyword(&mut self, k: Keyword) -> bool {
        match self.lexer.peek() {
            Ok(tok) => tok.is_keyword_kind(k),
            _ => false,
        }
    }

    fn ahead_is_symbol(&mut self, syb: Symbol) -> bool {
        match self.lexer.peek() {
            Ok(tok) => tok.is_symbol_kind(syb),
            _ => false,
        }
    }

    fn ahead_is_symbol_or(&mut self, s1: Symbol, s2: Symbol) -> bool {
        match self.lexer.peek() {
            Ok(tok) => tok.is_symbol_kind(s1) || tok.is_symbol_kind(s2),
            _ => false,
        }
    }

    fn new_expr(&mut self) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        self.lexer.advance();
        let callee = match self.ahead_is_keyword(Keyword::New) {
            true => self.new_expr(),
            false => self.member_expr(None),
        };
        let callee = match callee {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };
        let arguments = match self.ahead_is_symbol(Symbol::ParenL) {
            true => match self.args() {
                Ok(args) => args,
                Err(e) => return Err(e),
            },
            false => vec![],
        };
        loc.end = self.pos();
        let expr = NewExpr {
            loc,
            callee,
            arguments,
        };
        Ok(expr.into())
    }

    fn args(&mut self) -> Result<Vec<Expr>, ParsingError> {
        self.lexer.advance();
        let mut args: Vec<Expr> = vec![];
        loop {
            if self.ahead_is_symbol(Symbol::ParenR) {
                self.lexer.advance();
                break;
            }
            match self.assign_expr(false, None) {
                Ok(expr) => args.push(expr),
                Err(e) => return Err(e),
            };
            if self.ahead_is_symbol(Symbol::Comma) {
                self.lexer.advance();
            }
        }
        Ok(args)
    }

    fn call_expr(&mut self, callee: Option<Expr>) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let callee = match callee {
            Some(expr) => expr,
            _ => match self.member_expr(None) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            },
        };

        if !self.ahead_is_symbol(Symbol::ParenL) {
            return Ok(callee);
        }

        let arguments = match self.args() {
            Ok(args) => args,
            Err(e) => return Err(e),
        };
        loc.end = self.pos();
        let mut expr: Expr = CallExpr {
            loc,
            callee,
            arguments,
        }
        .into();
        if self.ahead_is_symbol_or(Symbol::Dot, Symbol::BracketL) {
            expr = match self.member_expr(Some(expr)) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            }
        } else if self.ahead_is_symbol(Symbol::ParenL) {
            expr = match self.call_expr(Some(expr)) {
                Ok(expr) => expr,
                Err(e) => return Err(e),
            }
        }
        Ok(expr)
    }

    fn member_expr(&mut self, obj: Option<Expr>) -> Result<Expr, ParsingError> {
        let mut loc = self.loc();
        let mut obj = match obj {
            Some(o) => o,
            _ => match self.primary_expr() {
                Ok(expr) => expr.into(),
                Err(e) => return Err(e),
            },
        };
        loop {
            if let Ok(tok) = self.lexer.peek() {
                if tok.is_symbol_kind(Symbol::Dot) {
                    self.lexer.advance();
                    match self.lexer.next() {
                        Ok(tok) => {
                            if !tok.is_id() {
                                return Err(ParserError::at_tok(&tok).into());
                            }
                            let prop = PrimaryExpr::Identifier(IdData::new(
                                tok.loc().clone(),
                                tok.id_data().clone().value,
                            ));
                            loc.end = self.pos();
                            obj = Expr::Member(Rc::new(MemberExpr {
                                loc,
                                object: obj,
                                property: prop.into(),
                                computed: false,
                            }));
                            continue;
                        }
                        Err(e) => return Err(e.into()),
                    }
                } else if tok.is_symbol_kind(Symbol::BracketL) {
                    self.lexer.advance();
                    let prop = match self.expr(false) {
                        Ok(expr) => expr,
                        Err(e) => return Err(e.into()),
                    };
                    match self.lexer.next() {
                        Ok(tok) => {
                            if !tok.is_symbol_kind(Symbol::BracketR) {
                                return Err(ParserError::at_tok(&tok).into());
                            }
                        }
                        Err(e) => return Err(e.into()),
                    }
                    loc.end = self.pos();
                    obj = Expr::Member(Rc::new(MemberExpr {
                        loc,
                        object: obj,
                        property: prop,
                        computed: true,
                    }));
                    continue;
                }
            }
            break;
        }
        Ok(obj)
    }

    fn primary_expr(&mut self) -> Result<PrimaryExpr, ParsingError> {
        match self.lexer.next() {
            Ok(tok) => {
                let loc = tok.loc().to_owned();
                if tok.is_keyword_kind(Keyword::This) {
                    Ok(ThisExprData::new(loc).into())
                } else if tok.is_id() {
                    Ok(IdData::new(loc, tok.id_data().value.clone()).into())
                } else if tok.is_undef() {
                    Ok(UndefData::new(loc).into())
                } else if tok.is_str() {
                    Ok(StringData::new(loc, tok.str_data().value.clone()).into())
                } else if tok.is_null() {
                    Ok(NullData::new(loc).into())
                } else if tok.is_bool() {
                    Ok(BoolData::new(loc, tok.bool_data().kind == BooleanLiteral::True).into())
                } else if tok.is_num() {
                    Ok(NumericData::new(loc, tok.num_data().value.clone()).into())
                } else if tok.is_symbol_kind(Symbol::BraceL) {
                    self.object_literal(&tok)
                } else if tok.is_symbol_kind(Symbol::BracketL) {
                    self.array_literal(&tok)
                } else if tok.is_symbol_kind(Symbol::ParenL) {
                    self.paren_expr(&tok)
                } else if tok.is_keyword_kind(Keyword::Function) {
                    self.fn_expr(&tok)
                } else {
                    Err(ParserError::at_tok(&tok).into())
                }
            }
            Err(e) => Err(e.into()),
        }
    }

    fn paren_expr(&mut self, open: &Token) -> Result<PrimaryExpr, ParsingError> {
        let mut loc = open.loc().to_owned();
        let value = match self.expr(false) {
            Ok(expr) => expr,
            Err(e) => return Err(e),
        };
        match self.ahead_is_symbol(Symbol::ParenR) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        }
        loc.end = self.pos();
        Ok(ParenData { loc, value }.into())
    }

    fn fn_expr(&mut self, open: &Token) -> Result<PrimaryExpr, ParsingError> {
        let mut loc = open.loc().to_owned();

        let mut id = None;
        if !self.ahead_is_symbol(Symbol::ParenL) {
            id = match self.primary_expr() {
                Ok(expr) => {
                    if !expr.is_id() {
                        return Err(ParserError::at(expr.loc()).into());
                    }
                    Some(expr)
                }
                Err(e) => return Err(e),
            }
        }

        match self.ahead_is_symbol(Symbol::ParenL) {
            true => self.lexer.advance(),
            false => return Err(ParserError::at(&self.loc()).into()),
        }

        let mut params = vec![];
        loop {
            if self.ahead_is_symbol(Symbol::ParenR) {
                self.lexer.advance();
                break;
            }
            match self.primary_expr() {
                Ok(e) => {
                    if !e.is_id() {
                        return Err(ParserError::at(e.loc()).into());
                    }
                    params.push(e);
                }
                Err(e) => return Err(e),
            }
            if self.ahead_is_symbol(Symbol::Comma) {
                self.lexer.advance();
            }
        }

        loc.end = self.pos();
        let body = match self.block_stmt() {
            Ok(stmt) => stmt,
            Err(e) => return Err(e),
        };

        let dec = FnDec {
            loc,
            id,
            params,
            body,
        };
        Ok(dec.into())
    }

    fn object_literal(&mut self, open: &Token) -> Result<PrimaryExpr, ParsingError> {
        let mut ret = ObjectData {
            loc: open.loc().to_owned(),
            properties: vec![],
        };
        loop {
            match self.lexer.peek() {
                Ok(tok) => {
                    if tok.is_symbol_kind(Symbol::BraceR) {
                        match self.lexer.next() {
                            Err(e) => return Err(e.into()),
                            _ => break,
                        }
                    } else if tok.is_symbol_kind(Symbol::Comma) {
                        match self.lexer.next() {
                            Err(e) => return Err(e.into()),
                            _ => (),
                        }
                    }
                    match self.object_prop() {
                        Ok(prop) => ret.properties.push(prop),
                        Err(e) => return Err(e),
                    }
                }
                Err(e) => return Err(e.into()),
            }
        }
        ret.loc.end = self.pos();
        Ok(PrimaryExpr::ObjectLiteral(ret))
    }

    fn object_prop(&mut self) -> Result<ObjectProperty, ParsingError> {
        let mut loc = self.loc();
        let key = match self.lexer.next() {
            Ok(tok) => {
                if tok.is_str() {
                    PrimaryExpr::Literal(Literal::String(StringData {
                        loc: tok.loc().to_owned(),
                        value: tok.str_data().value.clone(),
                    }))
                } else if tok.is_id() {
                    PrimaryExpr::Identifier(IdData {
                        loc: tok.loc().to_owned(),
                        name: tok.id_data().value.clone(),
                    })
                } else {
                    return Err(ParserError::at(tok.loc()).into());
                }
            }
            Err(e) => return Err(e.into()),
        };
        match self.lexer.next() {
            Ok(tok) => {
                if !tok.is_symbol_kind(Symbol::Colon) {
                    return Err(ParserError::at(tok.loc()).into());
                }
            }
            Err(e) => return Err(e.into()),
        }
        let value = match self.assign_expr(false, None) {
            Ok(expr) => expr,
            Err(e) => return Err(e.into()),
        };
        loc.end = self.pos();
        Ok(ObjectProperty {
            loc,
            key: key.into(),
            value,
        })
    }

    fn array_literal(&mut self, open: &Token) -> Result<PrimaryExpr, ParsingError> {
        let mut ret = ArrayData {
            loc: open.loc().to_owned(),
            value: vec![],
        };
        loop {
            match self.lexer.peek() {
                Ok(tok) => {
                    if tok.is_symbol_kind(Symbol::BracketR) {
                        match self.lexer.next() {
                            Err(e) => return Err(e.into()),
                            _ => break,
                        }
                    } else if tok.is_symbol_kind(Symbol::Comma) {
                        match self.lexer.next() {
                            Err(e) => return Err(e.into()),
                            _ => {
                                let expr = Expr::Primary(Rc::new(PrimaryExpr::Literal(
                                    Literal::Undef(UndefData {
                                        loc: tok.loc().clone(),
                                    }),
                                )));
                                ret.value.push(expr)
                            }
                        }
                    } else {
                        match self.assign_expr(false, None) {
                            Ok(expr) => ret.value.push(expr),
                            Err(e) => return Err(e.into()),
                        };
                        if self.ahead_is_symbol(Symbol::Comma) {
                            self.lexer.advance();
                        }
                    }
                }
                Err(e) => return Err(e.into()),
            }
        }
        Ok(ret.into())
    }

    fn pos(&mut self) -> Position {
        self.lexer.pos()
    }

    fn loc(&mut self) -> SourceLoc {
        self.lexer.loc()
    }
}

#[derive(Debug)]
pub struct ParserError {
    pub msg: String,
}

impl ParserError {
    pub fn new(msg: String) -> Self {
        ParserError { msg }
    }

    pub fn at(loc: &SourceLoc) -> Self {
        ParserError {
            msg: format!(
                "Unexpected token at line: {} column: {}",
                loc.start.line, loc.start.column
            ),
        }
    }

    pub fn at_tok(tok: &Token) -> Self {
        let loc = tok.loc();
        ParserError::at(loc)
    }

    pub fn default() -> Self {
        ParserError {
            msg: "".to_string(),
        }
    }
}

#[derive(Debug)]
pub enum ParsingError {
    Parser(ParserError),
    Lexer(LexError),
}

impl ParsingError {
    pub fn msg(&self) -> &str {
        match self {
            ParsingError::Parser(e) => e.msg.as_str(),
            ParsingError::Lexer(e) => e.msg.as_str(),
        }
    }
}

impl From<ParserError> for ParsingError {
    fn from(e: ParserError) -> Self {
        ParsingError::Parser(e)
    }
}

impl From<LexError> for ParsingError {
    fn from(e: LexError) -> Self {
        ParsingError::Lexer(e)
    }
}

```

## Visitor

为了在后端部分的方便，这里定义了一个访问器的 trait，他的目的是，保证后端访问时的 ast 类型。

```rust

pub trait AstVisitor<T, E> {
  fn prog(&mut self, prog: &Prog) -> Result<T, E>;

  fn stmt(&mut self, stmt: &Stmt) -> Result<T, E> {
    match stmt {
      Stmt::Block(s) => self.block_stmt(s),
      Stmt::VarDec(s) => self.var_dec_stmt(s),
      Stmt::Empty(s) => self.empty_stmt(s),
      Stmt::Expr(s) => self.expr_stmt(s),
      Stmt::If(s) => self.if_stmt(s),
      Stmt::For(s) => self.for_stmt(s),
      Stmt::ForIn(s) => self.for_in_stmt(s),
      Stmt::DoWhile(s) => self.do_while_stmt(s),
      Stmt::While(s) => self.while_stmt(s),
      Stmt::Cont(s) => self.cont_stmt(s),
      Stmt::Break(s) => self.break_stmt(s),
      Stmt::Return(s) => self.ret_stmt(s),
      Stmt::With(s) => self.with_stmt(s),
      Stmt::Switch(s) => self.switch_stmt(s),
      Stmt::Throw(s) => self.throw_stmt(s),
      Stmt::Try(s) => self.try_stmt(s),
      Stmt::Debugger(s) => self.debug_stmt(s),
      Stmt::Function(s) => self.fn_stmt(s),
    }
  }

  fn block_stmt(&mut self, stmt: &BlockStmt) -> Result<T, E>;
  fn var_dec_stmt(&mut self, stmt: &VarDec) -> Result<T, E>;
  fn empty_stmt(&mut self, stmt: &EmptyStmt) -> Result<T, E>;
  fn expr_stmt(&mut self, stmt: &ExprStmt) -> Result<T, E>;
  fn if_stmt(&mut self, stmt: &IfStmt) -> Result<T, E>;
  fn for_stmt(&mut self, stmt: &ForStmt) -> Result<T, E>;
  fn for_in_stmt(&mut self, stmt: &ForInStmt) -> Result<T, E>;
  fn do_while_stmt(&mut self, stmt: &DoWhileStmt) -> Result<T, E>;
  fn while_stmt(&mut self, stmt: &WhileStmt) -> Result<T, E>;
  fn cont_stmt(&mut self, stmt: &ContStmt) -> Result<T, E>;
  fn break_stmt(&mut self, stmt: &BreakStmt) -> Result<T, E>;
  fn ret_stmt(&mut self, stmt: &ReturnStmt) -> Result<T, E>;
  fn with_stmt(&mut self, stmt: &WithStmt) -> Result<T, E>;
  fn switch_stmt(&mut self, stmt: &SwitchStmt) -> Result<T, E>;
  fn throw_stmt(&mut self, stmt: &ThrowStmt) -> Result<T, E>;
  fn try_stmt(&mut self, stmt: &TryStmt) -> Result<T, E>;
  fn debug_stmt(&mut self, stmt: &DebugStmt) -> Result<T, E>;
  fn fn_stmt(&mut self, stmt: &FnDec) -> Result<T, E>;

  fn expr(&mut self, expr: &Expr) -> Result<T, E> {
    match expr {
      Expr::Primary(ex) => self.primary_expr(ex),
      Expr::Member(ex) => self.member_expr(ex),
      Expr::New(ex) => self.new_expr(ex),
      Expr::Call(ex) => self.call_expr(ex),
      Expr::Unary(ex) => self.unary_expr(ex),
      Expr::Binary(ex) => self.binary_expr(ex),
      Expr::Assignment(ex) => self.assign_expr(ex),
      Expr::Conditional(ex) => self.cond_expr(ex),
      Expr::Sequence(ex) => self.seq_expr(ex),
    }
  }

  fn member_expr(&mut self, expr: &MemberExpr) -> Result<T, E>;
  fn new_expr(&mut self, expr: &NewExpr) -> Result<T, E>;
  fn call_expr(&mut self, expr: &CallExpr) -> Result<T, E>;
  fn unary_expr(&mut self, expr: &UnaryExpr) -> Result<T, E>;
  fn binary_expr(&mut self, expr: &BinaryExpr) -> Result<T, E>;
  fn assign_expr(&mut self, expr: &AssignExpr) -> Result<T, E>;
  fn cond_expr(&mut self, expr: &CondExpr) -> Result<T, E>;
  fn seq_expr(&mut self, expr: &SeqExpr) -> Result<T, E>;
  fn primary_expr(&mut self, expr: &PrimaryExpr) -> Result<T, E> {
    match expr {
      PrimaryExpr::This(ex) => self.this_expr(ex),
      PrimaryExpr::Identifier(ex) => self.id_expr(ex),
      PrimaryExpr::Literal(ex) => self.literal(ex),
      PrimaryExpr::ArrayLiteral(ex) => self.array_literal(ex),
      PrimaryExpr::ObjectLiteral(ex) => self.object_literal(ex),
      PrimaryExpr::Parenthesized(ex) => self.paren_expr(ex),
      PrimaryExpr::Function(ex) => self.fn_expr(ex),
    }
  }

  fn this_expr(&mut self, expr: &ThisExprData) -> Result<T, E>;
  fn id_expr(&mut self, expr: &IdData) -> Result<T, E>;
  fn array_literal(&mut self, expr: &ArrayData) -> Result<T, E>;
  fn object_literal(&mut self, expr: &ObjectData) -> Result<T, E>;
  fn paren_expr(&mut self, expr: &ParenData) -> Result<T, E>;
  fn fn_expr(&mut self, expr: &FnDec) -> Result<T, E>;
  fn literal(&mut self, expr: &Literal) -> Result<T, E> {
    match expr {
      Literal::RegExp(ex) => self.regexp_expr(ex),
      Literal::Null(ex) => self.null_expr(ex),
      Literal::Undef(ex) => self.undef_expr(ex),
      Literal::String(ex) => self.str_expr(ex),
      Literal::Bool(ex) => self.bool_expr(ex),
      Literal::Numeric(ex) => self.num_expr(ex),
    }
  }

  fn regexp_expr(&mut self, expr: &RegExpData) -> Result<T, E>;
  fn null_expr(&mut self, expr: &NullData) -> Result<T, E>;
  fn undef_expr(&mut self, expr: &UndefData) -> Result<T, E>;
  fn str_expr(&mut self, expr: &StringData) -> Result<T, E>;
  fn bool_expr(&mut self, expr: &BoolData) -> Result<T, E>;
  fn num_expr(&mut self, expr: &NumericData) -> Result<T, E>;
}

```

# 代码生成

在编译器前端写完后，接下来我们要来实现我们的编译器后端，它包含代码生成和 vm 两个部分，我们先来实现我们的代码生成的代码，我们的 vm 将会使用寄存器+栈组件实现的，参考了 lua 的实现形式

## chunk

chunk 就是我们的代码块，他包含了源码转义后指令，从而操作栈和寄存器。

```rust
// 常量
#[derive(Debug, Clone)]
pub enum Const {
    String(String),
    Number(f64),
}

impl Const {
    pub fn new_str(s: &str) -> Self {
        Const::String(s.to_string())
    }

    pub fn new_num(n: f64) -> Self {
        Const::Number(n)
    }

    pub fn typ_id(&self) -> u8 {
        match self {
            Const::String(_) => 0,
            Const::Number(_) => 1,
        }
    }

    pub fn is_str(&self) -> bool {
        match self {
            Const::String(_) => true,
            _ => false,
        }
    }

    pub fn is_num(&self) -> bool {
        match self {
            Const::Number(_) => true,
            _ => false,
        }
    }

    pub fn num(&self) -> f64 {
        match self {
            Const::Number(v) => *v,
            _ => panic!(),
        }
    }

    pub fn str(&self) -> &str {
        match self {
            Const::String(v) => v.as_str(),
            _ => panic!(),
        }
    }

    pub fn eq(&self, c: &Const) -> bool {
        if self.typ_id() == c.typ_id() {
            let eq = match self {
                Const::Number(v) => *v == c.num(),
                Const::String(v) => v == c.str(),
            };
            return eq;
        }
        false
    }
}

// 一个带上upvalue的函数，是闭包，包含了名称、是否在堆栈上以及索引
#[derive(Debug, Clone)]
pub struct UpvalDesc {
    pub name: String,
    pub in_stack: bool,
    pub idx: u32,
}

// 变量
#[derive(Debug, Clone)]
pub struct Local {
    pub name: String,
}

/*
ABC 操作模式：
参数编码：这种模式的指令编码包括操作码、参数 A、参数 B、参数 C。
参数范围：参数 A、B、C 通常是 8 位（1 字节）的整数，因此每个参数的取值范围是 0 到 255。
用途：ABC 操作模式通常用于执行基本的操作，例如加载、存储、算术运算等。

ABx 操作模式：
参数编码：这种模式的指令编码包括操作码、参数 A 和参数 Bx。
参数范围：参数 A 通常是 8 位整数，而参数 Bx 通常是 18 位整数。参数 Bx 的范围更大，可以表示更大的常量索引或其他值。
用途：ABx 操作模式通常用于加载常量、跳转指令等需要较大参数范围的操作。

AsBx 操作模式：
参数编码：这种模式的指令编码包括操作码、参数 A 和参数 sBx。
参数范围：参数 A 通常是 8 位整数，而参数 sBx 通常是带符号的 18 位整数，允许正数和负数值。
用途：AsBx 操作模式通常用于有符号跳转指令，例如条件分支等。
*/
#[derive(Debug, Eq, PartialEq, Copy, Clone)]
pub enum OpMode {
    ABC,
    ABx,
    AsBx,
}

#[derive(Clone)]
pub struct Inst {
    pub raw: u32,
}

impl Inst {
    pub fn new() -> Self {
        Inst { raw: 0 }
    }

    pub fn new_abc(op: OpCode, a: u32, b: u32, c: u32) -> Self {
        let mut inst = Inst::new();
        inst.set_op(op);
        inst.set_a(a);
        inst.set_b(b);
        inst.set_c(c);
        inst
    }

    pub fn new_a_bx(op: OpCode, a: u32, bx: u32) -> Self {
        let mut inst = Inst::new();
        inst.set_op(op);
        inst.set_a(a);
        inst.set_bx(bx);
        inst
    }

    pub fn new_a_sbx(op: OpCode, a: u32, sbx: i32) -> Self {
        let mut inst = Inst::new();
        inst.set_op(op);
        inst.set_a(a);
        inst.set_sbx(sbx);
        inst
    }

    pub fn a(&self) -> u32 {
        // 右移 6 位 (self.raw >> 6)：这将 self.raw 中的参数 A 位移到最低有效位，以便提取。
        // 与 0xff (& 0xff)：它将除参数 A 位之外的其他位都清零，保留参数 A 的值。0xff 是一个 8 位二进制数，所有位都为 1，因此这个操作保留了参数 A 的 8 位值。
        (self.raw >> 6) & 0xff
    }

    pub fn b(&self) -> u32 {
        // 右移 23 位 (self.raw >> 23)：这将 self.raw 中的参数 B 位移到最低有效位，以便提取。
        // 与 0x1ff (& 0x1ff)：它将除参数 B 位之外的其他位都清零，保留参数 B 的值。0x1ff 是一个 9 位二进制数，所有位都为 1，因此这个操作保留了参数 B 的 9 位值。
        (self.raw >> 23) & 0x1ff
    }

    pub fn c(&self) -> u32 {
        // 右移 14 位 (self.raw >> 14)：这将 self.raw 中的参数 C 位移到最低有效位，以便提取。
        // 与 0x1ff (& 0x1ff)：它将除参数 C 位之外的其他位都清零，保留参数 C 的值。0x1ff 是一个 9 位二进制数，所有位都为 1，因此这个操作保留了参数 C 的 9 位值。
        (self.raw >> 14) & 0x1ff
    }

    pub fn bx(&self) -> u32 {
        // 右移 14 位 (self.raw >> 14)：这将 self.raw 中的参数 Bx 位移到最低有效位，以便提取。
        // 与 0x3ffff (& 0x3ffff)：它将除参数 Bx 位之外的其他位都清零，保留参数 Bx 的值。0x3ffff 是一个 18 位二进制数，所有位都为 1，因此这个操作保留了参数 Bx 的 18 位值。
        (self.raw >> 14) & 0x3ffff
    }

    pub fn sbx(&self) -> i32 {
        // 右移 14 位 (self.raw >> 14)：这将 self.raw 中的参数 sBx 位移到最低有效位，以便提取。
        // 与 0x3ffff (& 0x3ffff)：它将除参数 sBx 位之外的其他位都清零，保留参数 sBx 的值。0x3ffff 是一个 18 位二进制数，所有位都为 1，因此这个操作保留了参数 sBx 的 18 位值。
        // as i32：这将结果转换为带符号的 32 位整数。
        // - 131071：这个操作是将参数 sBx 转换为有符号整数，因为它是相对于 131071 的偏移值。
        let t = ((self.raw >> 14) & 0x3ffff) as i32;
        t - 131071
    }

    pub fn op(&self) -> u32 {
        // 与 0x3f (& 0x3f)：它将 self.raw 中的操作码位之外的其他位都清零，保留操作码的值。0x3f 是一个 6 位二进制数，所有位都为 1，因此这个操作保留了操作码的 6 位值。
        self.raw & 0x3f
    }

    pub fn set_op(&mut self, op: OpCode) {
        // !0x3f 表示对 0x3f 取反，即将 0x3f 中的所有位设为 0，只保留其他位不变。
        // | (op as u32)：将新的操作码值 op 添加到 self.raw 中，从而更新操作码。
        self.raw = (self.raw & !0x3f) | (op as u32);
    }

    pub fn set_a(&mut self, a: u32) {
        // 0xff << 6 表示将 0xff 左移 6 位，得到一个掩码，用于清除参数 A 的位。
        // !(0xff << 6) 对这个掩码取反，即将除参数 A 位之外的其他位都清零，保留参数 A 的位。
        // (a << 6) 将参数 A 的值左移 6 位，以匹配参数 A 在 self.raw 中的位置。
        self.raw = (self.raw & !(0xff << 6)) | (a << 6);
    }

    pub fn set_b(&mut self, b: u32) {
        // 0x1ff << 23 表示将 0x1ff 左移 23 位，得到一个掩码，用于清除参数 B 的位。
        // !(0x1ff << 23) 对这个掩码取反，即将除参数 B 位之外的其他位都清零，保留参数 B 的位。
        // (b << 23) 将参数 B 的值左移 23 位，以匹配参数 B 在 self.raw 中的位置。
        self.raw = (self.raw & !(0x1ff << 23)) | (b << 23);
    }

    pub fn set_c(&mut self, c: u32) {
        // 0x1ff << 14 表示将 0x1ff 左移 14 位，得到一个掩码，用于清除参数 C 的位。
        // !(0x1ff << 14) 对这个掩码取反，即将除参数 C 位之外的其他位都清零，保留参数 C 的位。
        // (c << 14) 将参数 C 的值左移 14 位，以匹配参数 C 在 self.raw 中的位置。
        self.raw = (self.raw & !(0x1ff << 14)) | (c << 14);
    }

    pub fn set_bx(&mut self, bx: u32) {
        // 0x3ffff << 14 表示将 0x3ffff 左移 14 位，得到一个掩码，用于清除参数 Bx 的位。
        // !(0x3ffff << 14) 对这个掩码取反，即将除参数 Bx 位之外的其他位都清零，保留参数 Bx 的位。
        // (bx << 14) 将参数 Bx 的值左移 14 位，以匹配参数 Bx 在 self.raw 中的位置。
        self.raw = (self.raw & !(0x3ffff << 14)) | (bx << 14);
    }

    pub fn set_sbx(&mut self, mut sbx: i32) {
        // sbx += 131071：这是一个将参数 sbx 增加 131071 的操作。它将参数 sbx 转换为无符号整数并增加 131071，因为虚拟机中的 sBx 参数通常是相对于 131071 的偏移值。
        sbx += 131071;
        let sbx = sbx as u32;
        // 0x3ffff << 14 表示将 0x3ffff 左移 14 位，得到一个掩码，用于清除参数 sbx 的位。
        // !(0x3ffff << 14) 对这个掩码取反，即将除参数 sbx 位之外的其他位都清零，保留参数 sbx 的位。
        // (sbx << 14) 将参数 sbx 的值左移 14 位，以匹配参数 sbx 在 self.raw 中的位置。
        self.raw = (self.raw & !(0x3ffff << 14)) | (sbx << 14);
    }

    // converts an integer to a "floating point byte"
    pub fn int2fb(mut x: u32) -> u32 {
        let mut e = 0; /* exponent */
        if x < 8 {
            return x;
        }
        while x >= 8 << 4 {
            /* coarse steps */
            x = (x + 0xf) >> 4; /* x = ceil(x / 16) */
            e += 4;
        }
        while x >= 8 << 1 {
            /* fine steps */
            x = (x + 1) >> 1; /* x = ceil(x / 2) */
            e += 1;
        }
        ((e + 1) << 3) | (x - 8)
    }

    pub fn fb2int(x: u32) -> u32 {
        if x < 8 {
            return x;
        }
        ((x & 7) + 8) << ((x >> 3) - 1)
    }
}

impl fmt::Debug for Inst {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let op = OpCode::from_u32(self.op());
        match op.mode() {
            OpMode::ABC => write!(
                f,
                "{:#?}{{ A: {}, B: {}, C: {} }}",
                op,
                self.a(),
                self.b(),
                self.c()
            ),
            OpMode::ABx => write!(f, "{:#?}{{ A: {}, Bx: {} }}", op, self.a(), self.bx()),
            OpMode::AsBx => write!(f, "{:#?}{{ A: {}, sBx: {} }}", op, self.a(), self.sbx()),
        }
    }
}
// 函数模板
#[derive(Debug, Clone)]
pub struct FnTpl {
    // 参数数量
    pub param_cnt: u8,
    // 是否支持可变参数
    pub is_vararg: bool,
    // 指令序列
    pub code: Vec<Inst>,
    pub consts: Vec<Const>,
    // 记录函数中引用的上值（外部变量）
    pub upvals: Vec<UpvalDesc>,
    // 局部变量
    pub locals: Vec<Local>,
    // 表示在函数内定义的子函数
    pub fun_tpls: Vec<FnTpl>,
}

impl FnTpl {
    pub fn new() -> Self {
        FnTpl {
            param_cnt: 0,
            is_vararg: false,
            code: vec![],
            consts: vec![],
            upvals: vec![],
            locals: vec![],
            fun_tpls: vec![],
        }
    }
}

// 代码块
#[derive(Debug)]
pub struct Chunk {
    pub sig: &'static str,
    pub ver: u64,
    pub upval_cnt: u8,
    pub fun_tpl: FnTpl,
}

#[derive(Debug, Eq, PartialEq, Copy, Clone, Hash)]
pub enum OpCode {
    MOVE,
    LOADK,
    LOADKX,
    LOADBOO,
    LOADNUL,
    LOADUNDEF,
    GETUPVAL,
    GETTABUP,
    GETTABLE,
    SETTABUP,
    SETUPVAL,
    SETTABLE,
    NEWTABLE,
    NEWARRAY,
    INITARRAY,
    THIS,
    ADD,
    SUB,
    MUL,
    MOD,
    DIV,
    LT,
    LE,
    EQ,
    EQS,
    JMP,
    TEST,
    TESTSET,
    BITAND,
    BITOR,
    BITXOR,
    SHL,
    SAR,
    SHR,
    UNM,
    NOT,
    BITNOT,
    CLOSURE,
    CALL,
    RETURN,
    NEW,
}

static mut OPCODE_NAME: Option<HashMap<OpCode, &'static str>> = None;
static mut OPCODE_MODE: Option<HashMap<OpCode, OpMode>> = None;

macro_rules! gen_opcode_map {
  ($($op:expr => $name:expr, $mode:expr)*) => {
    {
      let mut op_name = HashMap::new();
      let mut op_mode = HashMap::new();
      $(
        op_name.insert($op, $name);
        op_mode.insert($op, $mode);
      )*
      (Some(op_name), Some(op_mode))
    }
  };
}

fn init_opcodes() {
    let (op_name, op_mode) = gen_opcode_map! {
      OpCode::MOVE => "MOVE", OpMode::ABC
      OpCode::LOADK => "LOADK", OpMode::ABx
      OpCode::LOADKX => "LOADKX", OpMode::ABx
      OpCode::LOADBOO => "LOADBOO", OpMode::ABC
      OpCode::LOADNUL => "LOADNUL", OpMode::ABC
      OpCode::LOADUNDEF => "LOADUNDEF", OpMode::ABC
      OpCode::GETUPVAL => "GETUPVAL", OpMode::ABC
      OpCode::GETTABUP => "GETTABUP", OpMode::ABC
      OpCode::GETTABLE => "GETTABLE", OpMode::ABC
      OpCode::SETTABUP => "SETTABUP", OpMode::ABC
      OpCode::SETUPVAL => "SETUPVAL", OpMode::ABC
      OpCode::SETTABLE => "SETTABLE", OpMode::ABC
      OpCode::NEWTABLE => "NEWTABLE", OpMode::ABC
      OpCode::NEWARRAY => "NEWARRAY", OpMode::ABC
      OpCode::INITARRAY => "INITARRAY", OpMode::ABC
      OpCode::THIS => "THIS", OpMode::ABC
      OpCode::ADD => "ADD", OpMode::ABC
      OpCode::SUB => "SUB", OpMode::ABC
      OpCode::MUL => "MUL", OpMode::ABC
      OpCode::MOD => "MOD", OpMode::ABC
      OpCode::DIV => "DIV", OpMode::ABC
      OpCode::LT => "LT", OpMode::ABC
      OpCode::LE => "LE", OpMode::ABC
      OpCode::EQ => "EQ", OpMode::ABC
      OpCode::EQS => "EQS", OpMode::ABC
      OpCode::JMP => "JMP", OpMode::AsBx
      OpCode::TEST => "TEST", OpMode::ABC
      OpCode::TESTSET => "TESTSET", OpMode::ABC
      OpCode::BITAND => "BITAND", OpMode::ABC
      OpCode::BITOR => "BITOR", OpMode::ABC
      OpCode::BITXOR => "BITXOR", OpMode::ABC
      OpCode::SHL => "SHL", OpMode::ABC
      OpCode::SAR => "SAR", OpMode::ABC
      OpCode::SHR => "SHR", OpMode::ABC
      OpCode::UNM => "UNM", OpMode::ABC
      OpCode::NOT => "NOT", OpMode::ABC
      OpCode::BITNOT => "BITNOT", OpMode::ABC
      OpCode::CLOSURE => "CLOSURE", OpMode::ABC
      OpCode::CALL => "CALL", OpMode::ABC
      OpCode::RETURN => "RETURN", OpMode::ABC
      OpCode::NEW => "NEW", OpMode::ABC
    };
    unsafe {
        OPCODE_NAME = op_name;
        OPCODE_MODE = op_mode;
    }
}

static INIT_OPCODE_DATA_ONCE: Once = Once::new();
pub fn init_opcode_data() {
    INIT_OPCODE_DATA_ONCE.call_once(|| {
        init_opcodes();
    });
}

pub fn op_to_name(op: &OpCode) -> &'static str {
    unsafe { OPCODE_NAME.as_ref().unwrap().get(op).unwrap() }
}

pub fn op_to_mode(op: &OpCode) -> &'static OpMode {
    unsafe { OPCODE_MODE.as_ref().unwrap().get(op).unwrap() }
}

impl OpCode {
    pub fn from_u32(x: u32) -> Self {
        unsafe { transmute(x as u8) }
    }

    pub fn mode(&self) -> OpMode {
        *op_to_mode(self)
    }

    pub fn eq(&self, op: u32) -> bool {
        OpCode::from_u32(op) == *self
    }
}

```

## symtab

symtab 将会储存相关的调用环境，就像我们知道的一样，js 存在作用域和闭包，而 symtab 维护了不同作用域的寄存器及栈，从而保证在 chunk 执行时能准确地获取正确的数据。

```rust

pub type ScopePtr = *mut Scope;

pub fn as_scope(ptr: ScopePtr) -> &'static mut Scope {
    unsafe { &mut (*ptr) }
}

// 作用域
#[derive(Debug)]
pub struct Scope {
    pub id: usize,
    parent: ScopePtr,
    subs: Vec<ScopePtr>,
    pub params: HashSet<String>,
    pub bindings: LinkedHashSet<String>,
}

impl Scope {
    fn new(id: usize) -> ScopePtr {
        Box::into_raw(Box::new(Scope {
            id,
            parent: null_mut(),
            subs: vec![],
            params: HashSet::new(),
            bindings: LinkedHashSet::new(),
        }))
    }

    pub fn add_binding(&mut self, n: &str) {
        self.bindings.insert(n.to_string());
    }

    pub fn has_binding(&self, n: &str) -> bool {
        self.bindings.contains(n)
    }

    pub fn add_param(&mut self, n: &str) {
        self.params.insert(n.to_owned());
    }

    pub fn has_param(&self, n: &str) -> bool {
        self.params.contains(n)
    }
}

#[derive(Debug)]
pub struct SymTab {
    // 作用域个数
    i: usize,
    scopes: HashMap<usize, ScopePtr>,
    // 当前作用域
    s: ScopePtr,
}

impl SymTab {
    pub fn new() -> SymTab {
        let s = Scope::new(0);
        let mut scopes = HashMap::new();
        scopes.insert(as_scope(s).id, s);
        SymTab { i: 1, scopes, s }
    }

    pub fn enter_scope(&mut self) {
        let s = Scope::new(self.i);
        self.scopes.insert(self.i, s);
        self.i += 1;
        as_scope(s).parent = self.s;
        as_scope(self.s).subs.push(s);
        self.s = s;
    }

    pub fn leave_scope(&mut self) {
        self.s = as_scope(self.s).parent;
    }

    fn add_binding(&mut self, n: &str) {
        as_scope(self.s).add_binding(n);
    }

    fn add_param(&mut self, n: &str) {
        as_scope(self.s).add_param(n);
    }

    pub fn get_scope(&self, i: usize) -> ScopePtr {
        *self.scopes.get(&i).unwrap()
    }
}

impl Drop for SymTab {
    fn drop(&mut self) {
        self.scopes
            .values()
            .for_each(|s| unsafe { drop_in_place(*s) });
    }
}

impl AstVisitor<(), ()> for SymTab {
    fn prog(&mut self, prog: &Prog) -> Result<(), ()> {
        prog.body.iter().for_each(|s| self.stmt(s).unwrap());
        Ok(())
    }

    fn block_stmt(&mut self, stmt: &BlockStmt) -> Result<(), ()> {
        stmt.body.iter().for_each(|s| self.stmt(s).unwrap());
        Ok(())
    }

    fn var_dec_stmt(&mut self, stmt: &VarDec) -> Result<(), ()> {
        stmt.decs.iter().for_each(|dec| {
            self.add_binding(dec.id.id().name.as_str());
            if let Some(init) = &dec.init {
                self.expr(init).ok();
            }
        });
        Ok(())
    }

    fn empty_stmt(&mut self, _stmt: &EmptyStmt) -> Result<(), ()> {
        Ok(())
    }

    fn expr_stmt(&mut self, stmt: &ExprStmt) -> Result<(), ()> {
        self.expr(&stmt.expr).ok();
        Ok(())
    }

    fn if_stmt(&mut self, stmt: &IfStmt) -> Result<(), ()> {
        self.stmt(&stmt.cons).ok();
        if let Some(s) = &stmt.alt {
            self.stmt(s).ok();
        }
        Ok(())
    }

    fn for_stmt(&mut self, stmt: &ForStmt) -> Result<(), ()> {
        if let Some(init) = &stmt.init {
            match init {
                ForFirst::VarDec(dec) => self.var_dec_stmt(dec).unwrap(),
                _ => (),
            }
        }
        self.stmt(&stmt.body).ok();
        Ok(())
    }

    fn for_in_stmt(&mut self, stmt: &ForInStmt) -> Result<(), ()> {
        match &stmt.left {
            ForFirst::VarDec(dec) => self.var_dec_stmt(dec).unwrap(),
            _ => (),
        }
        self.stmt(&stmt.body).ok();
        Ok(())
    }

    fn do_while_stmt(&mut self, stmt: &DoWhileStmt) -> Result<(), ()> {
        self.stmt(&stmt.body).ok();
        Ok(())
    }

    fn while_stmt(&mut self, stmt: &WhileStmt) -> Result<(), ()> {
        self.stmt(&stmt.body).ok();
        Ok(())
    }

    fn cont_stmt(&mut self, _stmt: &ContStmt) -> Result<(), ()> {
        Ok(())
    }

    fn break_stmt(&mut self, _stmt: &BreakStmt) -> Result<(), ()> {
        Ok(())
    }

    fn ret_stmt(&mut self, stmt: &ReturnStmt) -> Result<(), ()> {
        if let Some(s) = &stmt.argument {
            self.expr(s).ok();
        }
        Ok(())
    }

    fn with_stmt(&mut self, _stmt: &WithStmt) -> Result<(), ()> {
        Ok(())
    }

    fn switch_stmt(&mut self, stmt: &SwitchStmt) -> Result<(), ()> {
        stmt.cases
            .iter()
            .for_each(|case| case.cons.iter().for_each(|s| self.stmt(s).unwrap()));
        Ok(())
    }

    fn throw_stmt(&mut self, stmt: &ThrowStmt) -> Result<(), ()> {
        Ok(())
    }

    fn try_stmt(&mut self, stmt: &TryStmt) -> Result<(), ()> {
        self.stmt(&stmt.block).ok();
        if let Some(h) = &stmt.handler {
            self.stmt(&h.body).ok();
        }
        if let Some(f) = &stmt.finalizer {
            self.stmt(&f).ok();
        }
        Ok(())
    }

    fn debug_stmt(&mut self, stmt: &DebugStmt) -> Result<(), ()> {
        Ok(())
    }

    fn fn_stmt(&mut self, stmt: &FnDec) -> Result<(), ()> {
        if let Some(id) = &stmt.id {
            let f_name = id.id().name.as_str();
            self.add_binding(f_name);
        }
        self.enter_scope();
        stmt.params.iter().for_each(|p| {
            let n = p.id().name.as_str();
            self.add_param(n);
            self.add_binding(n);
        });
        self.stmt(&stmt.body).ok();
        self.leave_scope();
        Ok(())
    }

    fn member_expr(&mut self, expr: &MemberExpr) -> Result<(), ()> {
        self.expr(&expr.object).ok();
        self.expr(&expr.property).ok();
        Ok(())
    }

    fn new_expr(&mut self, expr: &NewExpr) -> Result<(), ()> {
        self.expr(&expr.callee).ok();
        expr.arguments
            .iter()
            .for_each(|arg| self.expr(arg).unwrap());
        Ok(())
    }

    fn call_expr(&mut self, expr: &CallExpr) -> Result<(), ()> {
        self.expr(&expr.callee).ok();
        expr.arguments
            .iter()
            .for_each(|arg| self.expr(arg).unwrap());
        Ok(())
    }

    fn unary_expr(&mut self, expr: &UnaryExpr) -> Result<(), ()> {
        self.expr(&expr.argument).ok();
        Ok(())
    }

    fn binary_expr(&mut self, expr: &BinaryExpr) -> Result<(), ()> {
        self.expr(&expr.left).ok();
        self.expr(&expr.right).ok();
        Ok(())
    }

    fn assign_expr(&mut self, expr: &AssignExpr) -> Result<(), ()> {
        self.expr(&expr.left).ok();
        self.expr(&expr.right).ok();
        Ok(())
    }

    fn cond_expr(&mut self, expr: &CondExpr) -> Result<(), ()> {
        self.expr(&expr.test).ok();
        self.expr(&expr.cons).ok();
        self.expr(&expr.alt).ok();
        Ok(())
    }

    fn seq_expr(&mut self, expr: &SeqExpr) -> Result<(), ()> {
        expr.exprs.iter().for_each(|expr| self.expr(expr).unwrap());
        Ok(())
    }

    fn this_expr(&mut self, _expr: &ThisExprData) -> Result<(), ()> {
        Ok(())
    }

    fn id_expr(&mut self, _expr: &IdData) -> Result<(), ()> {
        Ok(())
    }

    fn array_literal(&mut self, expr: &ArrayData) -> Result<(), ()> {
        expr.value.iter().for_each(|expr| self.expr(expr).unwrap());
        Ok(())
    }

    fn object_literal(&mut self, expr: &ObjectData) -> Result<(), ()> {
        expr.properties.iter().for_each(|p| {
            self.expr(&p.key).ok();
            self.expr(&p.value).ok();
        });
        Ok(())
    }

    fn paren_expr(&mut self, expr: &ParenData) -> Result<(), ()> {
        self.expr(&expr.value).ok();
        Ok(())
    }

    fn fn_expr(&mut self, expr: &FnDec) -> Result<(), ()> {
        // `var a = function b() {}; b();` -> ReferenceError `b is not defined`
        self.enter_scope();
        expr.params.iter().for_each(|p| {
            let n = p.id().name.as_str();
            self.add_param(n);
            self.add_binding(n);
        });
        self.stmt(&expr.body).ok();
        self.leave_scope();
        Ok(())
    }

    fn regexp_expr(&mut self, _expr: &RegExpData) -> Result<(), ()> {
        Ok(())
    }

    fn null_expr(&mut self, _expr: &NullData) -> Result<(), ()> {
        Ok(())
    }

    fn undef_expr(&mut self, _expr: &UndefData) -> Result<(), ()> {
        Ok(())
    }

    fn str_expr(&mut self, _expr: &StringData) -> Result<(), ()> {
        Ok(())
    }

    fn bool_expr(&mut self, _expr: &BoolData) -> Result<(), ()> {
        Ok(())
    }

    fn num_expr(&mut self, _expr: &NumericData) -> Result<(), ()> {
        Ok(())
    }
}
```

## codegen

首先我们定义一些简单的指令信息：

```rust

pub type FnStatePtr = *mut FnState;

pub fn as_fn_state(ptr: FnStatePtr) -> &'static mut FnState {
    unsafe { &mut (*ptr) }
}

pub const ENV_NAME: &'static str = "__ENV__";

#[derive(Debug)]
pub struct LoopInfo {
    start: i32,
    end: i32,
    // index of break instructions in this loop
    brk: Vec<i32>,
    // index of continue instructions in this loop
    cont: Vec<i32>,
}

#[derive(Debug)]
pub struct FnState {
    id: usize,
    tpl: FnTpl,
    parent: FnStatePtr,
    idx_in_parent: u32,
    // 局部变量名称映射到寄存器
    local_reg_map: HashMap<String, u32>,
    subs: Vec<FnStatePtr>,
    // 可用的寄存器编号，用于分配新的寄存器
    free_reg: u32,
    // 结果寄存器
    res_reg: Vec<u32>,
    loop_stack: Vec<LoopInfo>,
}

fn fn_state_to_tpl(s: &FnState) -> FnTpl {
    let mut tpl = s.tpl.clone();
    s.subs.iter().for_each(|sp| {
        let st = fn_state_to_tpl(as_fn_state(*sp));
        tpl.fun_tpls.push(st);
    });
    tpl
}

impl FnState {
    pub fn new(id: usize) -> FnStatePtr {
        Box::into_raw(Box::new(FnState {
            id,
            tpl: FnTpl::new(),
            parent: null_mut(),
            idx_in_parent: 0,
            local_reg_map: HashMap::new(),
            subs: vec![],
            free_reg: 0,
            res_reg: vec![],
            loop_stack: vec![],
        }))
    }

    // 分配新的寄存器编号
    pub fn take_reg(&mut self) -> u32 {
        let r = self.free_reg;
        self.free_reg += 1;
        r
    }

    // 检查是否存在具有指定名称的局部变量
    pub fn has_local(&self, n: &str) -> bool {
        for v in &self.tpl.locals {
            if v.name.eq(n) {
                return true;
            }
        }
        false
    }

    // 声明一个新的局部变量并将其映射到寄存器
    pub fn def_local(&mut self, n: &str, reg: u32) {
        self.tpl.locals.push(Local {
            name: n.to_string(),
        });
        self.local_reg_map.insert(n.to_string(), reg);
    }

    // 获取指定局部变量名称对应的寄存器编号
    pub fn local2reg(&self, n: &str) -> u32 {
        *self.local_reg_map.get(n).unwrap()
    }

    // 检查是否存在具有指定名称的上值
    pub fn has_upval(&self, n: &str) -> bool {
        for uv in &self.tpl.upvals {
            if uv.name.eq(n) {
                return true;
            }
        }
        false
    }

    pub fn get_upval(&self, n: &str) -> Option<&UpvalDesc> {
        for uv in &self.tpl.upvals {
            if uv.name.eq(n) {
                return Some(uv);
            }
        }
        None
    }

    // 获取指定名称的上值在上值数组中的索引
    pub fn get_upval_idx(&self, n: &str) -> usize {
        self.tpl.upvals.iter().position(|uv| uv.name.eq(n)).unwrap()
    }

    pub fn add_upval(&mut self, n: &str) -> bool {
        if self.has_upval(n) {
            return true;
        }
        let parent = self.parent;
        if parent.is_null() {
            if !self.has_upval(ENV_NAME) {
                self.tpl.upvals.push(UpvalDesc {
                    name: ENV_NAME.to_string(),
                    in_stack: true,
                    idx: 0,
                });
            }
            return false;
        } else {
            let parent = as_fn_state(parent);
            if parent.has_local(n) {
                self.tpl.upvals.push(UpvalDesc {
                    name: n.to_string(),
                    in_stack: true,
                    idx: parent.local2reg(n),
                });
                return true;
            } else {
                let is_up = parent.add_upval(n);
                if is_up {
                    self.tpl.upvals.push(UpvalDesc {
                        name: n.to_string(),
                        in_stack: false,
                        idx: parent.get_upval_idx(n) as u32,
                    });
                    return true;
                } else {
                    self.add_upval(ENV_NAME);
                    return false;
                }
            }
        }
    }

    pub fn has_const(&self, c: &Const) -> bool {
        for c1 in &self.tpl.consts {
            if c1.eq(c) {
                return true;
            }
        }
        false
    }

    pub fn add_const(&mut self, c: &Const) {
        if self.has_const(c) {
            return;
        }
        self.tpl.consts.push(c.clone());
    }

    // 获取指定常量在常量数组中的索引
    pub fn const2idx(&self, c: &Const) -> usize {
        self.tpl.consts.iter().position(|c1| c1.eq(c)).unwrap()
    }

    // 追加一条指令
    pub fn append_inst(&mut self, inst: Inst) {
        self.tpl.code.push(inst);
    }

    fn push_res_reg(&mut self, r: u32) {
        self.res_reg.push(r);
    }

    fn pop_res_reg(&mut self) -> (u32, bool) {
        match self.res_reg.pop() {
            Some(r) => (r, false),
            None => (self.take_reg(), true),
        }
    }

    fn push_inst(&mut self, c: Inst) {
        self.tpl.code.push(c);
    }

    fn code_len(&self) -> i32 {
        self.tpl.code.len() as i32
    }

    // 添加一条跳转指令
    fn push_jmp(&mut self) -> i32 {
        let mut jmp = Inst::new();
        jmp.set_op(OpCode::JMP);
        jmp.set_sbx(self.code_len() + 1);
        self.push_inst(jmp);
        self.code_len() - 1
    }

    //  完成之前添加的跳转指令，设置其跳转目标
    fn fin_jmp(&mut self, idx: i32) {
        let cl = self.code_len();
        let jmp = self.tpl.code.get_mut(idx as usize).unwrap();
        jmp.set_sbx(cl - jmp.sbx());
    }

    // 完成之前添加的跳转指令，设置其跳转目标为给定的偏移值
    fn fin_jmp_sbx(&mut self, idx: i32, sbx: i32) {
        let jmp = self.tpl.code.get_mut(idx as usize).unwrap();
        jmp.set_sbx(sbx);
    }

    fn get_inst(&self, idx: i32) -> &Inst {
        self.tpl.code.get(idx as usize).unwrap()
    }

    // 设置可用寄存器编号的最大值，以便释放不再需要的寄存器
    fn free_reg_to(&mut self, r: u32) {
        self.free_reg = r;
    }

    fn free_regs(&mut self, rs: &[u32]) {
        if rs.len() > 0 {
            let mut mr = std::u32::MAX;
            for r in rs {
                mr = min(mr, *r);
            }
            self.free_reg_to(mr);
        }
    }

    fn get_sub(&mut self, idx: usize) -> &'static FnState {
        as_fn_state(self.subs[0])
    }

    fn enter_loop(&mut self) {
        self.loop_stack.push(LoopInfo {
            start: self.code_len(),
            end: 0,
            brk: vec![],
            cont: vec![],
        })
    }

    // 完成 break 指令的跳转目标
    fn fin_brk(&mut self) {
        if self.loop_stack.last().is_none() {
            return;
        }
        let info = self.loop_stack.last().unwrap();
        let end = info.end;
        for brk_idx in &info.brk {
            let jmp = self.get_inst(*brk_idx);
            let jmp_pc = jmp.sbx();
            let ptr = jmp as *const Inst as *mut Inst;
            unsafe {
                (*ptr).set_sbx(end - jmp_pc);
            }
        }
    }

    // 完成 continue 指令的跳转目标
    fn fin_cont(&mut self) {
        if self.loop_stack.last().is_none() {
            return;
        }
        let info = self.loop_stack.last().unwrap();
        let start = info.start;
        for cont_idx in &info.cont {
            let jmp = self.get_inst(*cont_idx);
            let jmp_pc = jmp.sbx();
            let ptr = jmp as *const Inst as *mut Inst;
            unsafe {
                (*ptr).set_sbx(start - jmp_pc);
            }
        }
    }

    // 跟踪循环的开始和结束
    fn leave_loop(&mut self) {
        self.loop_stack.last_mut().unwrap().end = self.code_len();
        self.fin_brk();
        self.fin_cont();
        self.loop_stack.pop();
    }
}

// RK 值（Register or Konstant值）是用于表示常量或寄存器的一种值。它的目的是将常量和寄存器引用都表示为RK值，从而减少指令中的操作数数量和字节码的大小。
pub fn kst_id_rk(id: usize) -> u32 {
    assert!(id < 256);
    (id | (1 << 8)) as u32
}

impl Drop for FnState {
    fn drop(&mut self) {
        for sub in &self.subs {
            unsafe {
                drop_in_place(*sub);
            }
        }
    }
}
```

接下来我们的 codegen 将会读取 ast，生成好对应的 chunk 及 symtab：

```rust
pub struct CodegenError {
    msg: String,
}

impl CodegenError {
    fn new(msg: &str) -> Self {
        CodegenError {
            msg: msg.to_string(),
        }
    }
}

#[derive(Debug)]
pub struct Codegen {
    scope_id_seed: usize,
    fs: FnStatePtr,
    symtab: SymTab,
}

impl Codegen {
    pub fn new(symtab: SymTab) -> Self {
        let fs = FnState::new(0);
        Codegen {
            scope_id_seed: 1,
            fs,
            symtab,
        }
    }

    pub fn gen(code: &str) -> Chunk {
        init_codegen_data();

        let code = String::from(code);
        let src = Source::new(&code);
        let mut lexer = Lexer::new(src);
        let mut parser = Parser::new(&mut lexer);
        let ast = match parser.prog() {
            Ok(node) => node,
            Err(e) => panic!("{}", e.msg().to_owned()),
        };

        let mut symtab = SymTab::new();
        symtab.prog(&ast).unwrap();

        let mut codegen = Codegen::new(symtab);
        codegen.prog(&ast).ok();

        let chk = Chunk {
            sig: "js",
            ver: 0x0001,
            upval_cnt: 0,
            fun_tpl: fn_state_to_tpl(codegen.fs_ref()),
        };

        chk
    }

    // 创建一个新的函数状态并将其设置为当前函数状态
    fn enter_fn_state(&mut self) {
        let cfs = self.fs_ref();
        let fs = as_fn_state(FnState::new(self.scope_id_seed));
        self.scope_id_seed += 1;
        fs.parent = self.fs;
        fs.idx_in_parent = cfs.subs.len() as u32;
        cfs.subs.push(fs);
        self.fs = fs;
    }

    fn leave_fn_state(&mut self) {
        self.fs = as_fn_state(self.fs).parent;
    }

    fn fs_ref(&self) -> &'static mut FnState {
        as_fn_state(self.fs)
    }

    fn symtab_scope(&self) -> &'static mut Scope {
        let fs = as_fn_state(self.fs);
        as_scope(self.symtab.get_scope(fs.id))
    }

    fn is_root_scope(&self) -> bool {
        self.symtab_scope().id == 0
    }

    fn has_binding(&self, n: &str) -> bool {
        self.symtab_scope().has_binding(n)
    }

    fn bindings(&self) -> &'static LinkedHashSet<String> {
        &self.symtab_scope().bindings
    }

    // 在当前作用域中声明所有绑定，通常在作用域的入口处调用
    fn declare_bindings(&mut self) {
        let fs = self.fs_ref();

        let bindings = self.bindings();
        if bindings.len() > 0 {
            let a = fs.take_reg() + self.symtab_scope().params.len() as u32;
            let mut b = 0;
            bindings.iter().for_each(|name| {
                fs.def_local(name.as_str(), b);
                b = fs.take_reg();
            });
            let mut inst = Inst::new();
            inst.set_op(OpCode::LOADUNDEF);
            inst.set_a(a);
            inst.set_b(b - 1);
            fs.push_inst(inst);
            fs.free_reg_to(b);
        }
    }

    // 处理一组语句，生成相应的字节码
    fn stmts(&mut self, stmts: &Vec<Stmt>) -> Result<(), CodegenError> {
        for stmt in stmts {
            match self.stmt(stmt) {
                Err(e) => return Err(e),
                _ => (),
            }
        }
        Ok(())
    }

    // 向当前函数状态的字节码中添加返回指令
    fn append_ret_inst(&mut self) {
        let mut ret = Inst::new();
        ret.set_op(OpCode::RETURN);
        ret.set_a(0);
        ret.set_b(1);
        self.fs_ref().push_inst(ret);
    }
}

impl Drop for Codegen {
    fn drop(&mut self) {
        unsafe {
            drop_in_place(self.fs);
        }
    }
}
// 将一个值设置为全局变量
fn new_set_global_inst(fs: &mut FnState, field_name: &str, val_reg: u32) -> Inst {
    let mut inst = Inst::new();
    inst.set_op(OpCode::SETTABUP);
    inst.set_a(fs.get_upval(ENV_NAME).unwrap().idx);
    let kst = Const::String(field_name.to_string());
    fs.add_const(&kst);
    inst.set_b(kst_id_rk(fs.const2idx(&kst)));
    inst.set_c(val_reg);
    inst
}

// 处理变量赋值操作,如果目标上值已经存在,生成 SETUPVAL 指令将值分配给上值,否则,会生成 SETTABUP 指令将值分配给表中的元素。
fn assign_upval(fs: &mut FnState, from_reg: u32, to_name: &str) -> Inst {
    let has_def = fs.add_upval(to_name);
    if has_def {
        let mut inst = Inst::new();
        inst.set_op(OpCode::SETUPVAL);
        inst.set_b(fs.get_upval(to_name).unwrap().idx);
        inst.set_a(from_reg);
        inst
    } else {
        let mut inst = Inst::new();
        inst.set_op(OpCode::SETTABUP);
        inst.set_a(fs.get_upval(ENV_NAME).unwrap().idx);
        let kst = Const::String(to_name.to_string());
        fs.add_const(&kst);
        inst.set_b(kst_id_rk(fs.const2idx(&kst)));
        inst.set_c(from_reg);
        inst
    }
}

// 如果在生成某个表达式或语句的过程中生成了一个 LOADK 指令，而后又发现它不再需要，就可以使用这个函数来弹出该指令，以减小生成字节码的冗余
fn pop_last_loadk(fs: &mut FnState) -> Option<u32> {
    let last_inst = fs.tpl.code.last().unwrap();
    let last_op = OpCode::from_u32(last_inst.op());
    if last_op == OpCode::LOADK {
        let last_inst = fs.tpl.code.pop().unwrap();
        Some(last_inst.bx())
    } else {
        None
    }
}

// 对stmt排序，进行函数声明提升，变量声明提升是通过调用declare_bindings实现的
fn hoist(stmts: &Vec<Stmt>) -> Vec<Stmt> {
    let mut not_fn = vec![];
    let mut fns = vec![];
    stmts.iter().for_each(|stmt| {
        if stmt.is_fn() {
            fns.push(stmt.clone());
        } else {
            not_fn.push(stmt.clone());
        }
    });
    fns.append(&mut not_fn);
    fns
}

impl AstVisitor<(), CodegenError> for Codegen {
    fn prog(&mut self, prog: &Prog) -> Result<(), CodegenError> {
        unsafe {
            if !IS_MOD_INITIALIZED {
                return Err(CodegenError::new(
                    "`init_codegen_data` should be called before this method",
                ));
            }
        }
        // 这里不使用self.declare_bindings()因为我们不能在根作用域中声明值，根作用域中的值只是全局对象字段的别名
        let stmts = hoist(&prog.body);
        match self.stmts(&stmts) {
            Err(e) => return Err(e),
            _ => (),
        }
        self.append_ret_inst();
        Ok(())
    }

    fn block_stmt(&mut self, stmt: &BlockStmt) -> Result<(), CodegenError> {
        let stmts = hoist(&stmt.body);
        self.stmts(&stmts)
    }

    fn var_dec_stmt(&mut self, stmt: &VarDec) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        stmt.decs.iter().for_each(|dec| {
            if let Some(init) = &dec.init {
                let n = dec.id.id().name.as_str();
                if fs.has_local(n) {
                    fs.push_res_reg(fs.local2reg(n));
                    self.expr(&init).ok();
                } else {
                    let mut tmp_regs = vec![];
                    let tmp_reg = fs.take_reg();
                    fs.push_res_reg(tmp_reg);
                    self.expr(&init).ok();
                    let rk = if let Some(r) = pop_last_loadk(fs) {
                        fs.free_reg_to(tmp_reg);
                        r
                    } else {
                        tmp_regs.push(tmp_reg);
                        tmp_reg
                    };

                    let inst = assign_upval(fs, rk, n);
                    fs.push_inst(inst);
                    fs.free_regs(&tmp_regs);
                }
            }
        });
        Ok(())
    }

    fn empty_stmt(&mut self, stmt: &EmptyStmt) -> Result<(), CodegenError> {
        Ok(())
    }

    fn expr_stmt(&mut self, stmt: &ExprStmt) -> Result<(), CodegenError> {
        self.expr(&stmt.expr).ok();
        Ok(())
    }

    fn if_stmt(&mut self, stmt: &IfStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let tr = fs.take_reg();
        fs.push_res_reg(tr);
        self.expr(&stmt.test).ok();

        let mut test = Inst::new();
        test.set_op(OpCode::TEST);
        test.set_a(tr);
        test.set_c(1);
        fs.push_inst(test);
        fs.free_reg_to(tr);

        // jmp to false
        let jmp1 = fs.push_jmp();
        self.stmt(&stmt.cons).ok();

        if let Some(alt) = &stmt.alt {
            // jmp over false
            let jmp2 = fs.push_jmp();
            fs.fin_jmp(jmp1);
            self.stmt(&alt).ok();
            fs.fin_jmp(jmp2);
        } else {
            fs.fin_jmp(jmp1);
        }

        Ok(())
    }

    fn for_stmt(&mut self, stmt: &ForStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        fs.enter_loop();

        if let Some(init) = &stmt.init {
            match init {
                ForFirst::VarDec(dec) => self.var_dec_stmt(dec).ok(),
                ForFirst::Expr(expr) => self.expr(expr).ok(),
            };
        }

        let s_len = fs.code_len();
        fs.loop_stack.last_mut().unwrap().start = s_len;

        let tr = fs.take_reg();
        if let Some(test) = &stmt.test {
            fs.push_res_reg(tr);
            self.expr(test).ok();
        } else {
            let mut b = Inst::new();
            b.set_op(OpCode::LOADBOO);
            b.set_a(tr);
            b.set_b(1);
            fs.push_inst(b);
        }

        let mut t = Inst::new();
        t.set_op(OpCode::TEST);
        t.set_a(tr);
        t.set_c(1);
        fs.push_inst(t);
        fs.free_reg_to(tr);

        // jmp out of loop
        let jmp1 = fs.push_jmp();

        self.stmt(&stmt.body).ok();

        if let Some(update) = &stmt.update {
            self.expr(update).ok();
        }

        // jmp to the loop start
        let jmp2 = fs.push_jmp();
        fs.fin_jmp(jmp1);

        let e_len = fs.code_len();
        fs.fin_jmp_sbx(jmp2, s_len - e_len);
        fs.leave_loop();
        Ok(())
    }

    fn for_in_stmt(&mut self, stmt: &ForInStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn do_while_stmt(&mut self, stmt: &DoWhileStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        fs.enter_loop();

        let s_len = fs.code_len();
        self.stmt(&stmt.body).ok();

        let tr = fs.take_reg();
        fs.push_res_reg(tr);
        self.expr(&stmt.test).ok();
        fs.free_reg_to(tr);

        let mut t = Inst::new();
        t.set_op(OpCode::TEST);
        t.set_a(tr);
        t.set_c(0);
        fs.push_inst(t);

        // jmp out of loop
        let jmp1 = fs.push_jmp();
        let e_len = fs.code_len();
        fs.fin_jmp_sbx(jmp1, s_len - e_len);
        fs.leave_loop();
        Ok(())
    }

    fn while_stmt(&mut self, stmt: &WhileStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        fs.enter_loop();

        let s_len = fs.code_len();

        let tr = fs.take_reg();
        fs.push_res_reg(tr);
        self.expr(&stmt.test).ok();
        fs.free_reg_to(tr);

        let mut t = Inst::new();
        t.set_op(OpCode::TEST);
        t.set_a(tr);
        t.set_c(1);
        fs.push_inst(t);

        // jmp out of loop
        let jmp1 = fs.push_jmp();
        self.stmt(&stmt.body).ok();

        // jmp to the loop start
        let jmp2 = fs.push_jmp();
        fs.fin_jmp(jmp1);

        let e_len = fs.code_len();
        fs.fin_jmp_sbx(jmp2, s_len - e_len);
        fs.leave_loop();
        Ok(())
    }

    fn cont_stmt(&mut self, stmt: &ContStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let len = fs.code_len();
        let mut inst = Inst::new();
        inst.set_op(OpCode::JMP);
        inst.set_sbx(len + 1);
        fs.push_inst(inst);
        fs.loop_stack.last_mut().unwrap().cont.push(len);
        Ok(())
    }

    fn break_stmt(&mut self, stmt: &BreakStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let len = fs.code_len();
        let mut inst = Inst::new();
        inst.set_op(OpCode::JMP);
        inst.set_sbx(len + 1);
        fs.push_inst(inst);
        fs.loop_stack.last_mut().unwrap().brk.push(len);
        Ok(())
    }

    fn ret_stmt(&mut self, stmt: &ReturnStmt) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut ret = Inst::new();
        ret.set_op(OpCode::RETURN);

        if let Some(arg) = &stmt.argument {
            let tr = fs.take_reg();
            fs.push_res_reg(tr);
            self.expr(arg).ok();
            ret.set_a(tr);
            ret.set_b(2);
            fs.free_reg_to(tr);
        }

        fs.push_inst(ret);
        Ok(())
    }

    fn with_stmt(&mut self, stmt: &WithStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn switch_stmt(&mut self, stmt: &SwitchStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn throw_stmt(&mut self, stmt: &ThrowStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn try_stmt(&mut self, stmt: &TryStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn debug_stmt(&mut self, stmt: &DebugStmt) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn fn_stmt(&mut self, stmt: &FnDec) -> Result<(), CodegenError> {
        // 没有 id 的函数是没有意义的，因为它不能稍后调用，所以在这种情况下我们可以跳过生成它的指令
        if stmt.id.is_none() {
            return Ok(());
        }

        let id = stmt.id.as_ref().unwrap().id().name.as_str();
        let pfs = self.fs_ref();

        self.enter_fn_state();
        let cfs = self.fs_ref();
        let ra = if pfs.has_local(id) {
            pfs.local2reg(id)
        } else {
            pfs.take_reg()
        };

        let mut inst = Inst::new();
        inst.set_op(OpCode::CLOSURE);
        inst.set_a(ra);
        inst.set_bx(cfs.idx_in_parent);
        pfs.push_inst(inst);

        if !pfs.has_local(id) {
            let inst = assign_upval(pfs, ra, id);
            pfs.push_inst(inst);
            pfs.free_reg_to(ra);
        }

        self.declare_bindings();
        self.stmt(&stmt.body).ok();
        self.append_ret_inst();

        self.leave_fn_state();
        Ok(())
    }

    fn member_expr(&mut self, expr: &MemberExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let mut inst = Inst::new();
        let tr = fs.take_reg();
        fs.push_res_reg(tr);
        self.expr(&expr.object).ok();
        tmp_regs.push(tr);

        inst.set_op(OpCode::GETTABLE);
        inst.set_a(res_reg);
        inst.set_b(tr);

        if expr.computed {
            let tr = fs.take_reg();
            fs.push_res_reg(tr);
            self.expr(&expr.property).ok();
            tmp_regs.push(tr);
            inst.set_c(tr);
        } else {
            let n = expr.property.primary().id().name.as_str();
            let kst = Const::new_str(n);
            fs.add_const(&kst);
            inst.set_c(kst_id_rk(fs.const2idx(&kst)));
        }

        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn new_expr(&mut self, expr: &NewExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let a = fs.take_reg();
        fs.push_res_reg(a);
        self.expr(&expr.callee).ok();
        tmp_regs.push(a);

        expr.arguments.iter().for_each(|arg| {
            let r = fs.take_reg();
            fs.push_res_reg(r);
            self.expr(arg).ok();
            tmp_regs.push(r);
        });

        let mut call = Inst::new();
        call.set_op(OpCode::NEW);
        call.set_a(a);
        call.set_b((expr.arguments.len() + 1) as u32);

        let ret_num = if is_tmp { 1 } else { 2 };
        call.set_c(ret_num);
        fs.push_inst(call);

        if !is_tmp && a != res_reg {
            let mut mov = Inst::new();
            mov.set_op(OpCode::MOVE);
            mov.set_a(res_reg);
            mov.set_b(a);
            fs.push_inst(mov);
        }
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn call_expr(&mut self, expr: &CallExpr) -> Result<(), CodegenError> {
        // 与 Lua 不同，JS 不支持多个返回值，也没有 VARARG（直到 ES5，ES6 中的 RestParameter 等于 Lua 中的 VARARG）。然而，这并不意味着 ES5 中 CALL 中的 B 不能为 0
        // 使用这种能力来实现 Function.prototype.apply

        let fs = self.fs_ref();
        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let a = fs.take_reg();
        fs.push_res_reg(a);
        self.expr(&expr.callee).ok();
        tmp_regs.push(a);

        expr.arguments.iter().for_each(|arg| {
            let r = fs.take_reg();
            fs.push_res_reg(r);
            self.expr(arg).ok();
            tmp_regs.push(r);
        });

        let mut call = Inst::new();
        call.set_op(OpCode::CALL);
        call.set_a(a);
        call.set_b((expr.arguments.len() + 1) as u32);

        let ret_num = if is_tmp { 1 } else { 2 };
        call.set_c(ret_num);
        fs.push_inst(call);

        if !is_tmp && a != res_reg {
            let mut mov = Inst::new();
            mov.set_op(OpCode::MOVE);
            mov.set_a(res_reg);
            mov.set_b(a);
            fs.push_inst(mov);
        }
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    // 处理一元表达式
    fn unary_expr(&mut self, expr: &UnaryExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let sym = expr.op.symbol_data().kind;
        match sym {
            Symbol::Sub | Symbol::Not | Symbol::BitNot => {
                let op = match sym {
                    Symbol::Sub => OpCode::UNM,
                    Symbol::Not => OpCode::NOT,
                    Symbol::BitNot => OpCode::BITNOT,
                    _ => panic!(),
                };

                let mut inst = Inst::new();
                inst.set_op(op);
                inst.set_a(res_reg);

                let tr = fs.take_reg();
                fs.push_res_reg(tr);
                self.expr(&expr.argument).ok();
                if let Some(r) = pop_last_loadk(fs) {
                    inst.set_b(r);
                } else {
                    inst.set_b(tr);
                }
                fs.push_inst(inst);
                fs.free_regs(&tmp_regs);
            }
            Symbol::Inc | Symbol::Dec => {
                let op_tok = if sym == Symbol::Inc {
                    Token::Symbol(SymbolData {
                        kind: Symbol::AssignAdd,
                        loc: SourceLoc::new(),
                    })
                } else {
                    Token::Symbol(SymbolData {
                        kind: Symbol::AssignSub,
                        loc: SourceLoc::new(),
                    })
                };

                if !expr.prefix {
                    // 如果 expr 是后缀，我们应该计算参数的值并将计算结果移至结果寄存器
                    let m_b = fs.take_reg();
                    fs.push_res_reg(m_b);
                    self.expr(&expr.argument).ok();
                    tmp_regs.push(m_b);

                    if !is_tmp {
                        let mut mov = Inst::new();
                        mov.set_op(OpCode::MOVE);
                        mov.set_a(res_reg);
                        mov.set_b(m_b);
                        fs.push_inst(mov);
                    }
                }

                let assign: Expr = AssignExpr {
                    loc: SourceLoc::new(),
                    op: op_tok,
                    left: expr.argument.clone(),
                    right: PrimaryExpr::Literal(Literal::Numeric(NumericData::new(
                        SourceLoc::new(),
                        "1".to_owned(),
                    )))
                    .into(),
                }
                .into();

                //  如果expr是前缀，我们应该将结果寄存器压入res_reg_stack，让匿名赋值将其结果放入其中
                if expr.prefix {
                    fs.push_res_reg(res_reg);
                }
                self.expr(&assign).ok();
                fs.free_regs(&tmp_regs);
            }
            _ => panic!(),
        }
        Ok(())
    }

    fn binary_expr(&mut self, expr: &BinaryExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let op_s = expr.op.symbol_data().kind;

        let mut inst = Inst::new();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        inst.set_a(res_reg);
        match symbol_to_opcode(op_s) {
            Some(op) => inst.set_op(*op),
            _ => unimplemented!(),
        }

        let r_lhs = fs.take_reg();
        fs.push_res_reg(r_lhs);
        self.expr(&expr.left).ok();
        if let Some(rb) = pop_last_loadk(fs) {
            inst.set_b(rb);
            fs.free_reg_to(r_lhs);
        } else {
            inst.set_b(r_lhs);
            tmp_regs.push(r_lhs);
        }

        let r_rhs = fs.take_reg();
        fs.push_res_reg(r_rhs);
        self.expr(&expr.right).ok();
        if let Some(rc) = pop_last_loadk(fs) {
            inst.set_c(rc);
            fs.free_reg_to(r_rhs);
        } else {
            inst.set_c(r_rhs);
            tmp_regs.push(r_rhs);
        }

        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);

        match op_s {
            Symbol::LT
            | Symbol::LE
            | Symbol::Eq
            | Symbol::EqStrict
            | Symbol::GT
            | Symbol::GE
            | Symbol::NotEq
            | Symbol::NotEqStrict => {
                match op_s {
                    Symbol::LT | Symbol::LE | Symbol::Eq | Symbol::EqStrict => {
                        fs.tpl.code.last_mut().unwrap().set_a(1)
                    }
                    _ => fs.tpl.code.last_mut().unwrap().set_a(0),
                }

                let mut lb1 = Inst::new();
                lb1.set_op(OpCode::LOADBOO);
                lb1.set_a(res_reg);
                lb1.set_b(1);
                lb1.set_c(1);
                fs.push_inst(lb1);

                let mut lb2 = Inst::new();
                lb2.set_op(OpCode::LOADBOO);
                lb2.set_a(res_reg);
                lb2.set_b(0);
                fs.push_inst(lb2);
            }
            Symbol::And | Symbol::Or => {
                let test_set = fs.tpl.code.last_mut().unwrap();
                let lhs_reg = test_set.b();
                let rhs_reg = test_set.c();
                test_set.set_c(if op_s == Symbol::And { 1 } else { 0 });

                let mut jmp = Inst::new();
                jmp.set_op(OpCode::JMP);
                jmp.set_sbx(1);
                fs.push_inst(jmp);

                let mut mov = Inst::new();
                mov.set_op(OpCode::MOVE);
                mov.set_a(res_reg);
                mov.set_b(rhs_reg);
                fs.push_inst(mov);
            }
            _ => (),
        }

        Ok(())
    }

    fn assign_expr(&mut self, expr: &AssignExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_res_tmp) = fs.pop_res_reg();
        if is_res_tmp {
            tmp_regs.push(res_reg)
        }

        let mut rc = fs.take_reg();
        fs.push_res_reg(rc);
        self.expr(&expr.right).ok();
        if let Some(r) = pop_last_loadk(fs) {
            fs.free_reg_to(rc);
            rc = r;
        } else {
            tmp_regs.push(rc);
        }

        let mut inst = Inst::new();
        match &expr.left {
            Expr::Primary(pri) => {
                if !pri.is_id() {
                    return Err(CodegenError::new("lhs must be LVal"));
                }
                let id = pri.id().name.as_str();
                if fs.has_local(id) {
                    inst.set_op(OpCode::MOVE);
                    inst.set_a(fs.local2reg(id));
                } else {
                    inst = assign_upval(fs, 0, id);
                }
            }
            Expr::Member(m) => {
                inst.set_op(OpCode::SETTABLE);
                let tr = fs.take_reg();
                fs.push_res_reg(tr);
                self.expr(&m.object).ok();
                inst.set_a(tr);
                tmp_regs.push(tr);

                if m.computed {
                    let rb = fs.take_reg();
                    fs.push_res_reg(rb);
                    self.expr(&m.property).ok();
                    tmp_regs.push(rb);
                    inst.set_b(rb);
                } else {
                    let n = m.property.primary().id().name.as_str();
                    let kst = Const::new_str(n);
                    fs.add_const(&kst);
                    inst.set_b(kst_id_rk(fs.const2idx(&kst)));
                }
            }
            _ => return Err(CodegenError::new("lhs must be LVal")),
        }

        // is `operator=`
        if !expr.op.is_symbol_kind(Symbol::Assign) {
            let rb = fs.take_reg();
            fs.push_res_reg(rb);
            self.expr(&expr.left).ok();
            tmp_regs.push(rb);

            let ra = fs.take_reg();
            let mut binop = Inst::new();
            binop.set_op(*symbol_to_opcode(expr.op.symbol_data().kind).unwrap());
            binop.set_a(ra);
            binop.set_b(rb);
            binop.set_c(rc);
            fs.push_inst(binop);
            tmp_regs.push(ra);
            rc = ra;
        }

        let op = OpCode::from_u32(inst.op());
        match op {
            OpCode::MOVE => inst.set_b(rc),
            OpCode::SETUPVAL => inst.set_a(rc),
            OpCode::SETTABLE | OpCode::SETTABUP => inst.set_c(rc),
            _ => panic!(),
        }
        fs.push_inst(inst);

        if !is_res_tmp {
            let mut mov = Inst::new();
            mov.set_op(OpCode::MOVE);
            mov.set_a(res_reg);
            mov.set_b(rc);
            fs.push_inst(mov);
        }
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn cond_expr(&mut self, expr: &CondExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let tr = fs.take_reg();
        fs.push_res_reg(tr);
        self.expr(&expr.test).ok();
        tmp_regs.push(tr);

        let mut test = Inst::new();
        test.set_op(OpCode::TESTSET);
        test.set_a(tr);
        test.set_c(0);
        fs.push_inst(test);

        let jmp1 = fs.push_jmp();
        fs.push_res_reg(res_reg);
        self.expr(&expr.cons).ok();

        let jmp2 = fs.push_jmp();
        fs.fin_jmp(jmp1);

        fs.push_res_reg(res_reg);
        self.expr(&expr.alt).ok();
        fs.fin_jmp(jmp2);

        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn seq_expr(&mut self, expr: &SeqExpr) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let len = expr.exprs.len();
        expr.exprs.iter().enumerate().for_each(|(i, expr)| {
            if i == len - 1 {
                fs.push_res_reg(res_reg);
            } else {
                let tmp_reg = fs.take_reg();
                fs.push_res_reg(tmp_reg);
                tmp_regs.push(tmp_reg);
            }
            self.expr(expr).ok();
        });
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn this_expr(&mut self, expr: &ThisExprData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }
        let mut this = Inst::new();
        this.set_op(OpCode::THIS);
        this.set_a(res_reg);
        fs.push_inst(this);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn id_expr(&mut self, expr: &IdData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let n = expr.name.as_str();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        if fs.has_local(n) {
            let mut inst = Inst::new();
            inst.set_op(OpCode::MOVE);
            inst.set_a(res_reg);
            inst.set_b(fs.local2reg(n));
            fs.push_inst(inst);
        } else {
            let has_def = fs.add_upval(n);
            if has_def {
                let mut inst = Inst::new();
                inst.set_op(OpCode::GETUPVAL);
                inst.set_a(res_reg);
                inst.set_b(fs.get_upval_idx(n) as u32);
                fs.push_inst(inst);
            } else {
                let kst = Const::String(n.to_string());
                fs.add_const(&kst);
                let mut inst = Inst::new();
                inst.set_op(OpCode::GETTABUP);
                inst.set_a(res_reg);
                inst.set_b(fs.get_upval(ENV_NAME).unwrap().idx);
                inst.set_c(kst_id_rk(fs.const2idx(&kst)));
                fs.push_inst(inst);
            }
        }

        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn array_literal(&mut self, expr: &ArrayData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let mut inst = Inst::new();
        inst.set_op(OpCode::NEWARRAY);

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        inst.set_a(res_reg);
        fs.push_inst(inst);

        if expr.value.len() > 0 {
            let mut i = 0;
            expr.value.iter().for_each(|expr| {
                let r = fs.take_reg();
                tmp_regs.push(r);
                fs.push_res_reg(r);
                self.expr(expr).ok();
                i += 1;
            });
            let mut inst = Inst::new();
            inst.set_op(OpCode::INITARRAY);
            inst.set_a(res_reg);
            inst.set_b(i);
            fs.push_inst(inst);
        }

        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn object_literal(&mut self, expr: &ObjectData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();
        let mut inst = Inst::new();
        inst.set_op(OpCode::NEWTABLE);

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        inst.set_a(res_reg);
        fs.push_inst(inst);

        if expr.properties.len() > 0 {
            expr.properties.iter().for_each(|prop| {
                let kst = if prop.key.primary().is_id() {
                    Const::String(prop.key.primary().id().name.clone())
                } else {
                    Const::String(prop.key.primary().literal().str().value.clone())
                };
                fs.add_const(&kst);

                let mut inst = Inst::new();
                inst.set_op(OpCode::SETTABLE);
                inst.set_a(res_reg);
                inst.set_b(kst_id_rk(fs.const2idx(&kst)));

                let r = fs.take_reg();
                tmp_regs.push(r);
                fs.push_res_reg(r);
                inst.set_c(r);

                self.expr(&prop.value).ok();
                fs.push_inst(inst);
            });
        }

        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn paren_expr(&mut self, expr: &ParenData) -> Result<(), CodegenError> {
        self.expr(&expr.value).ok();
        Ok(())
    }

    fn fn_expr(&mut self, expr: &FnDec) -> Result<(), CodegenError> {
        let pfs = self.fs_ref();
        let (res_reg, is_tmp) = pfs.pop_res_reg();
        assert!(!is_tmp);

        self.enter_fn_state();
        let cfs = self.fs_ref();

        let mut inst = Inst::new();
        inst.set_op(OpCode::CLOSURE);
        inst.set_a(res_reg);
        inst.set_bx(cfs.idx_in_parent);
        pfs.push_inst(inst);

        self.declare_bindings();
        self.stmt(&expr.body).ok();
        self.append_ret_inst();

        self.leave_fn_state();
        Ok(())
    }

    fn regexp_expr(&mut self, expr: &RegExpData) -> Result<(), CodegenError> {
        unimplemented!()
    }

    fn null_expr(&mut self, expr: &NullData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let mut inst = Inst::new();
        inst.set_op(OpCode::LOADNUL);
        inst.set_a(res_reg);
        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn undef_expr(&mut self, expr: &UndefData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let mut inst = Inst::new();
        inst.set_op(OpCode::LOADUNDEF);
        inst.set_a(res_reg);
        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn str_expr(&mut self, expr: &StringData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let kst = Const::String(expr.value.clone());
        fs.add_const(&kst);
        let mut inst = Inst::new();
        inst.set_op(OpCode::LOADK);
        inst.set_a(res_reg);
        inst.set_bx(kst_id_rk(fs.const2idx(&kst)));
        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn bool_expr(&mut self, expr: &BoolData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let mut inst = Inst::new();
        inst.set_op(OpCode::LOADBOO);
        inst.set_a(res_reg);
        inst.set_b(if expr.value { 1 } else { 0 });
        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }

    fn num_expr(&mut self, expr: &NumericData) -> Result<(), CodegenError> {
        let fs = self.fs_ref();

        let mut tmp_regs = vec![];
        let (res_reg, is_tmp) = fs.pop_res_reg();
        if is_tmp {
            tmp_regs.push(res_reg)
        }

        let kst = Const::Number(expr.value.parse().ok().unwrap());
        fs.add_const(&kst);
        let mut inst = Inst::new();
        inst.set_op(OpCode::LOADK);
        inst.set_a(res_reg);
        inst.set_bx(kst_id_rk(fs.const2idx(&kst)));
        fs.push_inst(inst);
        fs.free_regs(&tmp_regs);
        Ok(())
    }
}

static mut SYMBOL_OPCODE_MAP: Option<HashMap<Symbol, OpCode>> = None;

fn symbol_to_opcode(s: Symbol) -> Option<&'static OpCode> {
    unsafe { SYMBOL_OPCODE_MAP.as_ref().unwrap().get(&s) }
}

fn init_symbol_opcode_map() {
    let mut map = HashMap::new();
    map.insert(Symbol::Add, OpCode::ADD);
    map.insert(Symbol::Sub, OpCode::SUB);
    map.insert(Symbol::Mul, OpCode::MUL);
    map.insert(Symbol::Mod, OpCode::MOD);
    map.insert(Symbol::Div, OpCode::DIV);
    map.insert(Symbol::LT, OpCode::LT);
    map.insert(Symbol::GT, OpCode::LE);
    map.insert(Symbol::LE, OpCode::LE);
    map.insert(Symbol::GE, OpCode::LT);
    map.insert(Symbol::Eq, OpCode::EQ);
    map.insert(Symbol::NotEq, OpCode::EQ);
    map.insert(Symbol::EqStrict, OpCode::EQS);
    map.insert(Symbol::NotEqStrict, OpCode::EQS);
    map.insert(Symbol::BitAnd, OpCode::BITAND);
    map.insert(Symbol::BitOr, OpCode::BITOR);
    map.insert(Symbol::BitNot, OpCode::BITNOT);
    map.insert(Symbol::SHL, OpCode::SHL);
    map.insert(Symbol::SAR, OpCode::SAR);
    map.insert(Symbol::SHR, OpCode::SHR);
    map.insert(Symbol::And, OpCode::TESTSET);
    map.insert(Symbol::Or, OpCode::TESTSET);
    map.insert(Symbol::AssignAdd, OpCode::ADD);
    map.insert(Symbol::AssignSub, OpCode::SUB);
    map.insert(Symbol::AssignMul, OpCode::MUL);
    map.insert(Symbol::AssignDiv, OpCode::DIV);
    map.insert(Symbol::AssignMod, OpCode::MOD);
    map.insert(Symbol::AssignSHL, OpCode::SHL);
    map.insert(Symbol::AssignSAR, OpCode::SAR);
    map.insert(Symbol::AssignSHR, OpCode::SHR);
    map.insert(Symbol::AssignBitAnd, OpCode::BITAND);
    map.insert(Symbol::AssignBitOr, OpCode::BITOR);
    map.insert(Symbol::AssignBitXor, OpCode::BITXOR);
    unsafe {
        SYMBOL_OPCODE_MAP = Some(map);
    }
}

static mut IS_MOD_INITIALIZED: bool = false;

static INIT_CODEGEN_DATA_ONCE: Once = Once::new();
pub fn init_codegen_data() {
    INIT_CODEGEN_DATA_ONCE.call_once(|| {
        init_token_data();
        init_opcode_data();
        init_symbol_opcode_map();
        unsafe {
            IS_MOD_INITIALIZED = true;
        }
    });
}
```

其中这里的操作符（+ - \* / ..），我们这里使用了爬山算法，即，如果这个操作符的权重小于前一个，则将之前的先进行计算。

# vm

## gc

gc 在 js 引擎中是必不可少的，我们的实现是将每一个 js 的量都是包装成一个 GcObj，他将会记录引用计数，然后在 gc 执行时判断是否需要手动执行析构函数。

```rust

pub type GcObjPtr = *mut GcObj;
pub type JsObj = GcObj;
pub type JsObjPtr = GcObjPtr;

pub type JsStrPtr = *mut JsString;
pub type JsNumPtr = *mut JsNumber;
pub type JsBoolPtr = *mut JsBoolean;
pub type JsArrPtr = *mut JsArray;
pub type JsDictPtr = *mut JsDict;
pub type JsFunPtr = *mut JsFunction;

pub type UpValPtr = *mut UpVal;

pub type GcPtr = *mut Gc;

#[inline(always)]
pub fn as_obj_ptr<T>(ptr: *mut T) -> JsObjPtr {
    ptr as JsObjPtr
}

#[inline(always)]
pub fn as_obj<T>(ptr: *mut T) -> &'static mut GcObj {
    unsafe { &mut (*(ptr as GcObjPtr)) }
}

#[inline(always)]
pub fn as_str<T>(ptr: *mut T) -> &'static mut JsString {
    unsafe { &mut (*(ptr as JsStrPtr)) }
}

#[inline(always)]
pub fn as_num<T>(ptr: *mut T) -> &'static mut JsNumber {
    unsafe { &mut (*(ptr as JsNumPtr)) }
}

#[inline(always)]
pub fn as_bool<T>(ptr: *mut T) -> &'static mut JsBoolean {
    unsafe { &mut (*(ptr as JsBoolPtr)) }
}

#[inline(always)]
pub fn as_arr<T>(ptr: *mut T) -> &'static mut JsArray {
    unsafe { &mut (*(ptr as JsArrPtr)) }
}

#[inline(always)]
pub fn as_dict<T>(ptr: *mut T) -> &'static mut JsDict {
    unsafe { &mut (*(ptr as JsDictPtr)) }
}

#[inline(always)]
pub fn as_fun<T>(ptr: *mut T) -> &'static mut JsFunction {
    unsafe { &mut (*(ptr as JsFunPtr)) }
}

#[inline(always)]
pub fn as_uv<T>(ptr: *mut T) -> &'static mut UpVal {
    unsafe { &mut (*(ptr as UpValPtr)) }
}

#[inline(always)]
pub fn as_gc<T>(ptr: *mut T) -> &'static mut Gc {
    unsafe { &mut (*(ptr as GcPtr)) }
}

#[repr(C)]
#[derive(Debug, Eq, PartialEq, Copy, Clone)]
pub enum GcObjKind {
    String,
    Number,

    Array,
    Dict,
    Function,

    UpVal,

    Boolean,
    Null,
    Undef,
}

pub type Deinit = fn(ptr: JsObjPtr);

fn default_deinit(ptr: JsObjPtr) {}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct GcObj {
    // 引用计数
    ref_cnt: usize,
    pub kind: GcObjKind,
    gc: GcPtr,
    // 手动析构函数
    deinit: Deinit,
}

impl GcObj {
    pub fn inc(&mut self) {
        self.ref_cnt += 1;
    }

    pub fn dec(&mut self) {
        as_gc(self.gc).dec(self);
    }

    pub fn gc(&self) -> GcPtr {
        self.gc
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsString {
    base: GcObj,
    pub d: String,
}

fn js_str_deinit(ptr: JsObjPtr) {
    let r = as_str(ptr);
    let s = mem::size_of_val(r);
    unsafe {
        drop_in_place(r);
    }
    as_gc(r.base.gc).heap_size -= s;
}

impl JsString {
    pub fn new(gc: &mut Gc, is_root: bool) -> JsStrPtr {
        let ptr = Box::into_raw(Box::new(JsString {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::String,
                gc,
                deinit: js_str_deinit,
            },
            d: "".to_owned(),
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsNumber {
    base: GcObj,
    pub d: f64,
}

fn js_num_deinit(ptr: JsObjPtr) {
    let r = as_num(ptr);
    let s = mem::size_of_val(r);
    unsafe {
        drop_in_place(r);
    }
    as_gc(r.base.gc).heap_size -= s;
}

impl JsNumber {
    fn new(gc: &mut Gc, is_root: bool) -> JsNumPtr {
        let ptr = Box::into_raw(Box::new(JsNumber {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Number,
                deinit: js_num_deinit,
                gc,
            },
            d: 0.0,
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsBoolean {
    base: GcObj,
    pub d: bool,
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsArray {
    base: GcObj,
    d: Vec<JsObjPtr>,
}

fn js_arr_deinit(ptr: JsObjPtr) {
    let r = as_arr(ptr);
    let s = mem::size_of_val(r);

    let gc = as_gc(r.base.gc);
    for pp in &r.d {
        gc.dec(*pp);
    }

    unsafe {
        drop_in_place(r);
    }
    gc.heap_size -= s;
}

impl JsArray {
    fn new(gc: &mut Gc, is_root: bool) -> JsArrPtr {
        let ptr = Box::into_raw(Box::new(JsArray {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Array,
                deinit: js_arr_deinit,
                gc,
            },
            d: vec![],
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsDict {
    base: GcObj,
    pub d: HashMap<String, JsObjPtr>,
}

fn js_dict_deinit(ptr: JsObjPtr) {
    let r = as_dict(ptr);
    let s = mem::size_of_val(r);

    let gc = as_gc(r.base.gc);
    for (_, pp) in &r.d {
        gc.dec(*pp);
    }

    unsafe {
        drop_in_place(r);
    }
    gc.heap_size -= s;
}

impl JsDict {
    fn new(gc: &mut Gc, is_root: bool) -> JsDictPtr {
        let ptr = Box::into_raw(Box::new(JsDict {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Dict,
                deinit: js_dict_deinit,
                gc,
            },
            d: HashMap::new(),
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }

    pub fn insert(&mut self, k: &str, v: JsObjPtr) {
        self.d.insert(k.to_owned(), v);
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct UpVal {
    base: GcObj,
    pub v: JsObjPtr,
}

fn upval_deinit(ptr: JsObjPtr) {
    let r = as_uv(ptr);
    let s = mem::size_of_val(r);

    let gc = as_gc(r.base.gc);
    gc.dec(r.v);

    unsafe {
        drop_in_place(r);
    }
    gc.heap_size -= s;
}

impl UpVal {
    fn new(gc: &mut Gc, v: JsObjPtr, is_root: bool) -> UpValPtr {
        as_obj(v).inc();
        let ptr = Box::into_raw(Box::new(UpVal {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::UpVal,
                deinit: upval_deinit,
                gc,
            },
            v,
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct JsFunction {
    base: GcObj,
    pub f: *const c_void,
    pub is_native: bool,
    pub upvals: Vec<UpValPtr>,
    pub prototype: JsObjPtr,
}

fn js_fun_deinit(ptr: JsObjPtr) {
    let r = as_fun(ptr);
    let s = mem::size_of_val(r);

    r.upvals.iter().for_each(|uv| as_obj(*uv).dec());

    unsafe {
        drop_in_place(r);
    }
    as_gc(r.base.gc).heap_size -= s;
}

impl JsFunction {
    fn new(gc: &mut Gc, is_root: bool) -> JsFunPtr {
        let ptr = Box::into_raw(Box::new(JsFunction {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Function,
                deinit: js_fun_deinit,
                gc,
            },
            f: null(),
            is_native: false,
            upvals: vec![],
            prototype: null_mut(),
        }));
        gc.register(as_obj_ptr(ptr), is_root);
        ptr
    }
}

#[repr(C)]
#[derive(Debug)]
pub struct Gc {
    heap_size: usize,
    max_heap_size: usize,
    // 存储对象指针，以便进行垃圾回收
    obj_list: LinkedHashSet<GcObjPtr>,
    // 用于存储根对象的指针，以确保它们不被垃圾回收
    roots: HashSet<GcObjPtr>,
    // 存储标记阶段中需要标记的对象
    marked: Vec<GcObjPtr>,
    js_null_: JsObjPtr,
    js_undef_: JsObjPtr,
    js_true_: JsBoolPtr,
    js_false_: JsBoolPtr,
}

static INIT_GC_DATA_ONCE: Once = Once::new();

impl Gc {
    pub fn new(max_heap_size: usize) -> Box<Self> {
        let mut gc = Box::new(Gc {
            heap_size: 0,
            max_heap_size,
            obj_list: LinkedHashSet::new(),
            roots: HashSet::new(),
            marked: vec![],
            js_null_: null_mut(),
            js_undef_: null_mut(),
            js_true_: null_mut(),
            js_false_: null_mut(),
        });
        gc.init_data();
        gc
    }

    fn init_data(&mut self) {
        self.js_null_ = Box::into_raw(Box::new(GcObj {
            ref_cnt: 1,
            kind: GcObjKind::Null,
            gc: self as GcPtr,
            deinit: default_deinit,
        }));

        self.js_undef_ = Box::into_raw(Box::new(GcObj {
            ref_cnt: 1,
            kind: GcObjKind::Undef,
            gc: self as GcPtr,
            deinit: default_deinit,
        }));

        self.js_true_ = Box::into_raw(Box::new(JsBoolean {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Boolean,
                gc: self as GcPtr,
                deinit: default_deinit,
            },
            d: true,
        }));

        self.js_false_ = Box::into_raw(Box::new(JsBoolean {
            base: GcObj {
                ref_cnt: 1,
                kind: GcObjKind::Boolean,
                gc: self as GcPtr,
                deinit: default_deinit,
            },
            d: false,
        }));
    }

    pub fn js_null(&self) -> JsObjPtr {
        self.js_null_
    }

    pub fn js_undef(&self) -> JsObjPtr {
        self.js_undef_
    }

    pub fn js_true(&self) -> JsBoolPtr {
        self.js_true_
    }

    pub fn js_false(&self) -> JsBoolPtr {
        self.js_false_
    }

    pub fn new_str(&mut self, is_root: bool) -> JsStrPtr {
        let need = mem::size_of::<JsString>();
        self.xgc(need);
        self.heap_size += need;
        JsString::new(self, is_root)
    }

    pub fn new_str_from_kst(&mut self, kst: &Const, is_root: bool) -> JsStrPtr {
        let s = self.new_str(is_root);
        as_str(s).d = kst.str().to_owned();
        s
    }

    pub fn new_obj_from_kst(&mut self, kst: &Const, is_root: bool) -> JsObjPtr {
        match kst {
            Const::String(_) => as_obj_ptr(self.new_str_from_kst(kst, is_root)),
            Const::Number(_) => as_obj_ptr(self.new_num_from_kst(kst, is_root)),
        }
    }

    pub fn new_num(&mut self, is_root: bool) -> JsNumPtr {
        let need = mem::size_of::<JsNumber>();
        self.xgc(need);
        self.heap_size += need;
        JsNumber::new(self, is_root)
    }

    pub fn new_num_from_kst(&mut self, kst: &Const, is_root: bool) -> JsNumPtr {
        let n = self.new_num(is_root);
        as_num(n).d = kst.num();
        n
    }

    pub fn new_arr(&mut self, is_root: bool) -> JsArrPtr {
        let need = mem::size_of::<JsArray>();
        self.xgc(need);
        self.heap_size += need;
        JsArray::new(self, is_root)
    }

    pub fn new_dict(&mut self, is_root: bool) -> JsDictPtr {
        let need = mem::size_of::<JsDict>();
        self.xgc(need);
        self.heap_size += need;
        JsDict::new(self, is_root)
    }

    pub fn new_fun(&mut self, is_root: bool) -> JsFunPtr {
        let need = mem::size_of::<JsFunction>();
        self.xgc(need);
        self.heap_size += need;
        JsFunction::new(self, is_root)
    }

    pub fn new_upval(&mut self, v: JsObjPtr, is_root: bool) -> UpValPtr {
        let need = mem::size_of::<UpVal>();
        self.xgc(need);
        self.heap_size += need;
        UpVal::new(self, v, is_root)
    }

    pub fn register(&mut self, ptr: JsObjPtr, is_root: bool) {
        self.obj_list.insert(ptr);
        if is_root {
            self.roots.insert(ptr);
        }
    }

    pub fn remove_root(&mut self, ptr: JsObjPtr) {
        self.roots.remove(&as_obj_ptr(ptr));
    }

    pub fn append_root(&mut self, ptr: JsObjPtr) {
        self.roots.insert(ptr);
    }

    pub fn inc<T>(&self, ptr: *mut T) {
        as_obj(ptr).ref_cnt += 1;
    }

    pub fn dec<T>(&mut self, ptr: *mut T) {
        let ptr = ptr as JsObjPtr;
        if !self.obj_list.contains(&ptr) {
            return;
        }

        let obj = as_obj(ptr);
        match obj.kind {
            GcObjKind::Null | GcObjKind::Undef | GcObjKind::Boolean => return,
            _ => (),
        }
        let em = format!("dropping dangling object {:#?} {:#?}", ptr, obj);
        assert!(obj.ref_cnt > 0, "{}", em);
        obj.ref_cnt -= 1;
        if obj.ref_cnt == 0 {
            self.drop(ptr);
        }
    }

    pub fn drop<T>(&mut self, ptr: *mut T) {
        let ptr = ptr as JsObjPtr;
        let obj = as_obj(ptr);
        self.obj_list.remove(&ptr);
        self.roots.remove(&ptr);
        (obj.deinit)(obj);
    }

    pub fn xgc(&mut self, need: usize) {
        let s = self.heap_size + need;
        if s >= self.max_heap_size {
            self.gc();
            assert!(self.heap_size + need <= self.max_heap_size);
        }
    }

    fn reset_all_ref_cnt(&self) {
        for pp in &self.obj_list {
            as_obj(*pp).ref_cnt = 0;
        }
    }

    fn mark_phase(&mut self) {
        // 标记根对象
        for root in &self.roots {
            self.marked.push(*root);
        }
        // 递归标记
        while let Some(ptr) = self.marked.pop() {
            let obj = as_obj(ptr);
            obj.ref_cnt += 1;
            if obj.ref_cnt == 1 {
                match obj.kind {
                    GcObjKind::String
                    | GcObjKind::Number
                    | GcObjKind::Boolean
                    | GcObjKind::Undef
                    | GcObjKind::Null => (),
                    GcObjKind::Array => {
                        for pp in &as_arr(obj).d {
                            self.marked.push(*pp)
                        }
                    }
                    GcObjKind::Dict => {
                        for (_, pp) in &as_dict(obj).d {
                            self.marked.push(*pp)
                        }
                    }
                    GcObjKind::UpVal => {
                        self.marked.push(as_uv(obj).v);
                    }
                    GcObjKind::Function => {
                        for pp in &as_fun(obj).upvals {
                            self.marked.push(as_obj_ptr(*pp))
                        }
                    }
                }
            }
        }
    }

    // 垃圾回收
    fn sweep_phase(&mut self) {
        let mut cc = vec![];
        for pp in &self.obj_list {
            let ptr = *pp;
            if as_obj(ptr).ref_cnt == 0 {
                cc.push(ptr);
            }
        }
        cc.iter().for_each(|ptr| {
            self.drop(*ptr);
        });
    }

    pub fn gc(&mut self) {
        self.reset_all_ref_cnt();
        self.mark_phase();
        self.sweep_phase();
    }
}

impl Drop for Gc {
    fn drop(&mut self) {
        for p in self.obj_list.clone() {
            if self.obj_list.contains(&p) {
                self.dec(p);
            }
        }
    }
}

pub struct LocalScope {
    vals: HashSet<JsObjPtr>,
}

impl LocalScope {
    pub fn new() -> Self {
        LocalScope {
            vals: HashSet::new(),
        }
    }

    pub fn reg<T>(&mut self, v: *mut T) -> JsObjPtr {
        let v = v as JsObjPtr;
        self.vals.insert(v);
        v
    }
}

impl Drop for LocalScope {
    fn drop(&mut self) {
        self.vals.iter().for_each(|v| as_obj(*v).dec())
    }
}
```

## GcObj 相关方法

```rust
impl GcObj {
    // 判断对象是否支持通过值传递,字符串或数字,它们支持通过值传递
    pub fn pass_by_value(&self) -> bool {
        match self.kind {
            GcObjKind::String | GcObjKind::Number => true,
            _ => false,
        }
    }

    // 是否可以强制转换
    pub fn check_coercible(&self) -> bool {
        match self.kind {
            GcObjKind::Undef | GcObjKind::Null => false,
            _ => true,
        }
    }

    // 值传递，将对象的值复制到一个新对象中
    pub fn x_pass_by_value(&mut self) -> JsObjPtr {
        let gc = as_gc(self.gc());
        match self.kind {
            GcObjKind::String => {
                let v = gc.new_str(false);
                as_str(v).d = as_str(self).d.clone();
                as_obj_ptr(v)
            }
            GcObjKind::Number => {
                let v = gc.new_num(false);
                as_num(v).d = as_num(self).d;
                as_obj_ptr(v)
            }
            _ => panic!(),
        }
    }

    pub fn eqs_true(&mut self) -> bool {
        self.kind == GcObjKind::Boolean && as_bool(self).d
    }

    pub fn eqs_false(&mut self) -> bool {
        self.kind == GcObjKind::Boolean && !as_bool(self).d
    }

    // ToPrimitive
    pub fn t_pri(&mut self) -> JsObjPtr {
        as_obj(self).inc();
        as_obj_ptr(self)
    }

    // ToNumber
    pub fn t_num(&mut self) -> JsNumPtr {
        let gc = as_gc(self.gc());
        match self.kind {
            GcObjKind::Undef => {
                let n = gc.new_num(false);
                as_num(n).d = f64::NAN;
                n
            }
            GcObjKind::Null => gc.new_num(false),
            GcObjKind::Boolean => {
                let n = gc.new_num(false);
                let is_true = as_obj(self).eqs_true();
                as_num(n).d = if is_true { 1.0 } else { 0.0 };
                n
            }
            GcObjKind::Number => {
                self.inc();
                as_obj_ptr(self) as JsNumPtr
            }
            GcObjKind::String => {
                let n = gc.new_num(false);
                as_num(n).set_v_str(as_str(self));
                n
            }
            _ => as_obj(self.t_pri()).t_num(),
        }
    }

    // ToBoolean
    pub fn t_bool(&mut self) -> JsBoolPtr {
        let gc = as_gc(self.gc());
        match self.kind {
            GcObjKind::Undef => gc.js_false(),
            GcObjKind::Null => gc.js_false(),
            GcObjKind::Boolean => as_obj_ptr(self) as JsBoolPtr,
            GcObjKind::Number => {
                let n = as_num(self);
                if n.d == 0.0 || n.d.is_nan() {
                    return gc.js_false();
                }
                gc.js_true()
            }
            GcObjKind::String => {
                let s = as_str(self);
                if s.d.len() == 0 {
                    return gc.js_false();
                }
                gc.js_true()
            }
            _ => gc.js_true(),
        }
    }

    // 是否为非基本类型
    pub fn is_compound(&self) -> bool {
        match self.kind {
            GcObjKind::String
            | GcObjKind::Number
            | GcObjKind::Boolean
            | GcObjKind::Null
            | GcObjKind::Undef => false,
            _ => true,
        }
    }

    pub fn eq(a: JsObjPtr, b: JsObjPtr) -> bool {
        let gc = as_gc(as_obj(a).gc());
        let mut local_scope = LocalScope::new();
        let ta = as_obj(a).kind;
        let tb = as_obj(b).kind;

        if ta == tb {
            if ta == GcObjKind::Undef {
                return true;
            } else if ta == GcObjKind::Null {
                return true;
            } else if ta == GcObjKind::Number {
                let av = as_num(a).d;
                let bv = as_num(b).d;
                if av.is_nan() {
                    return false;
                }
                if bv.is_nan() {
                    return false;
                }
                return av == bv;
            } else if ta == GcObjKind::String {
                let av = &as_str(a).d;
                let bv = &as_str(b).d;
                return av == bv;
            } else if ta == GcObjKind::Boolean {
                return a == b;
            }
        } else {
            if ta == GcObjKind::Undef && tb == GcObjKind::Null
                || ta == GcObjKind::Null && tb == GcObjKind::Undef
            {
                return true;
            }
            if ta == GcObjKind::Number && tb == GcObjKind::String {
                let nb = gc.new_num(false);
                local_scope.reg(nb);
                as_num(nb).set_v_str(as_str(b));
                return as_num(a).eq(nb);
            }
            if ta == GcObjKind::String && tb == GcObjKind::Number {
                let na = gc.new_num(false);
                local_scope.reg(na);
                as_num(na).set_v_str(as_str(a));
                return as_num(b).eq(na);
            }
            if ta == GcObjKind::Boolean {
                let na = gc.new_num(false);
                local_scope.reg(na);
                as_num(na).set_v_bool(a);
                return GcObj::eq(as_obj_ptr(na), b);
            }
            if tb == GcObjKind::Boolean {
                let nb = gc.new_num(false);
                local_scope.reg(nb);
                as_num(nb).set_v_bool(b);
                return GcObj::eq(a, as_obj_ptr(nb));
            }
            if (ta == GcObjKind::Number || ta == GcObjKind::String) && as_obj(b).is_compound() {
                let pri = as_obj(b).t_pri();
                local_scope.reg(pri);
                return GcObj::eq(a, pri);
            }
            if (tb == GcObjKind::Number || tb == GcObjKind::String) && as_obj(a).is_compound() {
                let pri = as_obj(a).t_pri();
                local_scope.reg(pri);
                return GcObj::eq(b, pri);
            }
        }
        return false;
    }

    pub fn lt(a: JsObjPtr, b: JsObjPtr) -> bool {
        let mut local_scope = LocalScope::new();
        let gc = as_gc(as_obj(a).gc());

        let pa = as_obj(a).t_pri();
        local_scope.reg(a);

        let pb = as_obj(b).t_pri();
        local_scope.reg(b);

        let ta = as_obj(pa).kind;
        let tb = as_obj(pb).kind;

        if !(ta == GcObjKind::String && tb == GcObjKind::String) {
            let na = as_obj(pa).t_num();
            local_scope.reg(na);
            let nb = as_obj(pb).t_num();
            local_scope.reg(nb);

            // NaN 的任何比较都会返回 false
            // 这返回false以保持LE只返回一种类型值
            if as_num(na).d.is_nan() {
                // return gc.js_undef();
                return false;
            }
            if as_num(nb).d.is_nan() {
                // return gc.js_undef();
                return false;
            }
            if as_num(na).d == as_num(nb).d {
                return false;
            }
            if as_num(na).d < as_num(nb).d {
                return true;
            }
            return false;
        }

        let sa = as_str(pa);
        let sb = as_str(pb);
        if sa.d.starts_with(sb.d.as_str()) {
            return false;
        }
        if sb.d.starts_with(sa.d.as_str()) {
            return true;
        }

        let cbs = sb.d.chars().collect::<Vec<_>>();
        for ca in sa.d.chars().enumerate() {
            let cb = *cbs.get(ca.0).unwrap();
            if ca.1 < cb {
                return true;
            } else if ca.1 > cb {
                return false;
            }
        }
        return false;
    }

    pub fn le(a: JsObjPtr, b: JsObjPtr) -> bool {
        GcObj::lt(a, b) || GcObj::eq(a, b)
    }
}



impl JsNumber {
  pub fn add(&mut self, b: JsNumPtr) -> JsNumPtr {
    let gc = as_gc(as_obj(self).gc());
    let n = gc.new_num(false);
    let a = self.d;
    let b = as_num(b).d;
    as_num(n).d = a + b;
    n
  }

  pub fn sub(&mut self, b: JsNumPtr) -> JsNumPtr {
    let gc = as_gc(as_obj(self).gc());
    let n = gc.new_num(false);
    let a = self.d;
    let b = as_num(b).d;
    as_num(n).d = a - b;
    n
  }

  pub fn mul(&mut self, b: JsNumPtr) -> JsNumPtr {
    let gc = as_gc(as_obj(self).gc());
    let n = gc.new_num(false);
    let a = self.d;
    let b = as_num(b).d;
    as_num(n).d = a * b;
    n
  }

  pub fn div(&mut self, b: JsNumPtr) -> JsNumPtr {
    let gc = as_gc(as_obj(self).gc());
    let n = gc.new_num(false);
    let a = self.d;
    let b = as_num(b).d;
    as_num(n).d = a / b;
    n
  }

  pub fn modulo(&mut self, b: JsNumPtr) -> JsNumPtr {
    let gc = as_gc(as_obj(self).gc());
    let n = gc.new_num(false);
    let a = self.d;
    let b = as_num(b).d;
    as_num(n).d = a % b;
    n
  }

  pub fn eq(&mut self, b: JsNumPtr) -> bool {
    let a = self.d;
    let b = as_num(b).d;
    a == b
  }

  pub fn set_v_str(&mut self, b: JsStrPtr) {
    self.d = match as_str(b).d.parse().ok() {
      Some(v) => v,
      _ => f64::NAN,
    }
  }

  pub fn set_v_bool(&mut self, b: JsObjPtr) {
    let gc = as_gc(as_obj(self).gc());
    let b = b as JsBoolPtr;
    self.d = if b == gc.js_true() { 1.0 } else { 0.0 }
  }
}

impl JsDict {
  pub fn set(&mut self, k: JsObjPtr, v: JsObjPtr) {
    let k = as_str(k).d.as_str();
    as_obj(v).inc();
    match self.d.get(k) {
      Some(old) => as_obj(*old).dec(),
      _ => (),
    }
    self.d.insert(k.to_owned(), v);
  }

  pub fn get(&mut self, k: JsObjPtr) -> JsObjPtr {
    let gc = as_gc(as_obj(self).gc());
    let k = as_str(k).d.as_str();
    match self.d.get(k) {
      Some(v) => *v,
      None => gc.js_undef(),
    }
  }
}

```

## exec

```rust

#[derive(Debug)]
pub struct RuntimeError {
    pub msg: String,
}

impl RuntimeError {
    fn new(msg: &str) -> Self {
        RuntimeError {
            msg: msg.to_owned(),
        }
    }
}

pub type CallInfoPtr = *mut CallInfo;

#[inline(always)]
pub fn as_ci<T>(ptr: *mut T) -> &'static mut CallInfo {
    unsafe { &mut (*(ptr as CallInfoPtr)) }
}

#[repr(C)]
#[derive(Debug, Clone)]
pub struct CallInfo {
    prev: CallInfoPtr,
    next: CallInfoPtr,
    pc: u32,
    fun: u32,
    base: u32,
    is_native: bool,
    open_upvals: HashMap<u32, UpValPtr>,
    is_new: bool,
    this: JsObjPtr,
}

impl CallInfo {
    pub fn new() -> CallInfoPtr {
        Box::into_raw(Box::new(CallInfo {
            prev: null_mut(),
            next: null_mut(),
            pc: 0,
            fun: 0,
            base: 0,
            is_native: false,
            open_upvals: HashMap::new(),
            is_new: false,
            this: null_mut(),
        }))
    }

    pub fn set_this(&mut self, this: JsObjPtr) {
        if !self.this.is_null() {
            as_obj(self.this).dec();
        }
        if !this.is_null() {
            as_obj(this).inc();
        }
        self.this = this;
    }
}

impl Drop for CallInfo {
    fn drop(&mut self) {
        if !self.this.is_null() {
            as_obj(self.this).dec();
        }
    }
}

pub enum RKValue {
    Kst(&'static Const),
    JsObj(JsObjPtr),
}

pub type VmPtr = *mut Vm;

#[inline(always)]
pub fn as_vm<T>(ptr: *mut T) -> &'static mut Vm {
    unsafe { &mut (*(ptr as VmPtr)) }
}

pub struct Vm {
    gc: Box<Gc>,
    c: Chunk,
    ci: CallInfoPtr,
    s: Vec<GcObjPtr>,
    pub env: JsDictPtr,
}

impl JsFunction {
    fn tpl(&self) -> &FnTpl {
        unsafe { &(*(self.f as *const FnTpl)) }
    }
}

impl Gc {
    pub fn new_fun_native(&mut self, f: NativeFn, is_root: bool) -> JsFunPtr {
        let nf = self.new_fun(is_root);
        as_fun(nf).is_native = true;
        as_fun(nf).f = f as *const c_void;
        nf
    }
}

pub type NativeFn = fn(vm: VmPtr);

fn print_obj(vm: VmPtr) {
    let args = as_vm(vm).get_args();
    args.iter().for_each(|arg| match as_obj(*arg).kind {
        GcObjKind::Undef => println!("undefined"),
        GcObjKind::Number => println!("{:#?}", as_num(*arg).d),
        GcObjKind::String => println!("{:#?}", as_str(*arg).d),
        GcObjKind::Function => println!("{:#?}", as_fun(*arg)),
        _ => (),
    });
}

impl Vm {
    pub fn new(c: Chunk, max_heap_size: usize) -> Box<Self> {
        let mut v = Box::new(Vm {
            gc: Gc::new(max_heap_size),
            c,
            ci: CallInfo::new(),
            s: vec![],
            env: null_mut(),
        });
        v.env = v.gc.new_dict(true);

        let print_fn = v.gc.new_fun(false);
        as_fun(print_fn).is_native = true;
        as_fun(print_fn).f = print_obj as *const c_void;
        as_obj(print_fn).inc();
        as_dict(v.env).insert("print", as_obj_ptr(print_fn));

        v.s.push(v.env as GcObjPtr);

        let f = v.gc.new_fun(true);
        let fr = as_fun(f);
        fr.f = &v.c.fun_tpl as *const FnTpl as *const c_void;
        fr.is_native = false;
        v.s.push(f as GcObjPtr);

        let uv = v.gc.new_upval(as_obj_ptr(v.env), false);
        as_ci(v.ci).fun = 1;
        as_ci(v.ci).base = 2;
        as_ci(v.ci).open_upvals.insert(0, uv);

        as_obj(uv).inc();
        fr.upvals.insert(0, uv);

        v
    }

    fn set_stack_slot(&mut self, i: u32, v: JsObjPtr) {
        self.gc.inc(v);
        let i = i as usize;
        // 之前栈上已经有一个旧值，则需要适当地处理旧值，包括从根集中移除和减少旧值的引用计数
        if i > self.s.len() - 1 {
            self.s.resize(i + 1, self.gc.js_undef());
        }
        match self.s.get(i) {
            Some(old) => {
                if !(*old).is_null() {
                    self.gc.remove_root(*old);
                    self.gc.dec(*old);
                }
            }
            _ => (),
        }
        self.gc.append_root(v);
        self.s[i] = v;
    }

    fn get_fn(&self) -> Option<JsFunPtr> {
        if self.ci.is_null() {
            return None;
        }
        let fi = as_ci(self.ci).fun as usize;
        match self.s.get(fi) {
            Some(f) => Some(*f as JsFunPtr),
            _ => None,
        }
    }

    fn fetch(&mut self) -> Result<Option<&Inst>, RuntimeError> {
        let f = match self.get_fn() {
            Some(f) => f,
            _ => return Ok(None),
        };
        let f = as_fun(f);
        if f.is_native {
            return Err(RuntimeError::new("using native fun as js"));
        }
        let pc = as_ci(self.ci).pc as usize;
        as_ci(self.ci).pc += 1;
        let code = &f.tpl().code;
        Ok(code.get(pc))
    }

    pub fn exec(&mut self) -> Result<(), RuntimeError> {
        loop {
            let i = match self.fetch() {
                Err(e) => return Err(e),
                Ok(inst) => match inst {
                    None => break,
                    Some(i) => i.clone(),
                },
            };
            match self.dispatch(i) {
                Err(e) => return Err(e),
                Ok(_) => (),
            }
        }
        Ok(())
    }

    fn get_kst(&self, r: u32) -> Option<&'static Const> {
        let r = r as usize;
        let mask = 1 << 8;
        let is_k = r & mask == 256;
        if !is_k {
            return None;
        }
        let f = self.get_fn().unwrap();
        let cs = &as_fun(f).tpl().consts;
        Some(&cs[r - 256])
    }

    fn rk(&self, base: u32, r: u32) -> RKValue {
        if let Some(k) = self.get_kst(r) {
            return RKValue::Kst(k);
        }
        RKValue::JsObj(self.get_stack_item(base + r))
    }

    fn x_rk(&mut self, base: u32, r: u32) -> (JsObjPtr, bool) {
        let mut is_new = false;
        let v = match self.rk(base, r) {
            RKValue::Kst(kst) => {
                is_new = true;
                self.gc.new_obj_from_kst(kst, false)
            }
            RKValue::JsObj(k) => k,
        };
        (v, is_new)
    }

    pub fn get_args(&self) -> Vec<JsObjPtr> {
        let ci = as_ci(self.ci);
        let len = ci.base - ci.fun - 1;
        let mut args = vec![];
        for i in 1..=len {
            args.push(self.get_stack_item(ci.fun + i));
        }
        args
    }

    fn set_return(&mut self, ret: JsObjPtr) {
        self.set_stack_slot(as_ci(self.ci).fun, ret);
    }

    fn get_stack_item(&self, i: u32) -> JsObjPtr {
        match self.s.get(i as usize) {
            Some(v) => {
                if v.is_null() {
                    self.gc.js_undef()
                } else {
                    *v
                }
            }
            _ => self.gc.js_undef(),
        }
    }

    fn get_upval(&self, i: u32) -> UpValPtr {
        let f = as_fun(self.get_fn().unwrap());
        *f.upvals.get(i as usize).unwrap()
    }

    fn get_fn_tpl(&self, i: u32) -> *const FnTpl {
        let cf = self.get_fn().unwrap();
        let tpl = &as_fun(cf).tpl().fun_tpls[i as usize];
        tpl as *const FnTpl
    }

    fn find_upvals(&mut self, uv_desc: &UpvalDesc) -> UpValPtr {
        let ci = as_ci(self.ci);
        let i = ci.base + uv_desc.idx;
        let open = match as_ci(self.ci).open_upvals.get(&i) {
            Some(uv) => *uv,
            None => {
                let v = self.get_stack_item(i);
                let uv = self.gc.new_upval(v, false);
                as_ci(self.ci).open_upvals.insert(i, uv);
                uv
            }
        };
        open
    }

    fn close_upvals(&mut self) {
        let ci = as_ci(self.ci);
        if ci.is_native {
            return;
        }
        for (i, uv) in &ci.open_upvals {
            as_obj(ci.open_upvals[&i]).dec();
        }
        ci.open_upvals.clear();
    }

    fn clean_ci_stack(&mut self, ret_num: u32) {
        let f = as_ci(self.ci).fun + ret_num;
        let len = self.s.len() as u32;
        for i in f..len {
            self.set_stack_slot(i, self.gc.js_undef());
        }
    }

    fn post_call(&mut self, ret_num: u32) {
        self.close_upvals();
        self.clean_ci_stack(ret_num);

        let cci = self.ci;
        let pci = as_ci(cci).prev;
        if !pci.is_null() {
            as_ci(pci).next = null_mut();
        }
        self.ci = pci;
        unsafe {
            drop_in_place(cci);
        }
    }

    fn dispatch(&mut self, i: Inst) -> Result<(), RuntimeError> {
        let op = OpCode::from_u32(i.op());
        match op {
            OpCode::GETTABUP => {
                let mut ls = LocalScope::new();
                let ra = i.a();
                let rc = i.c();
                let base = as_ci(self.ci).base;
                let (k, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(k);
                }
                let uv = as_uv(self.get_upval(i.b()));
                let v = as_dict(uv.v).get(k);
                self.set_stack_slot(as_ci(self.ci).base + ra, v);
            }
            OpCode::SETTABUP => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;
                let (k, is_new) = self.x_rk(base, i.b());
                if is_new {
                    ls.reg(k);
                }
                let (v, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(v);
                }
                let uv = as_uv(self.get_upval(i.a()));
                as_dict(uv.v).set(k, v);
            }
            OpCode::LOADK => {
                let mut ls = LocalScope::new();
                let ra = i.a();
                let (v, is_new) = self.x_rk(0, i.bx());
                if is_new {
                    ls.reg(v);
                }
                self.set_stack_slot(as_ci(self.ci).base + ra, v);
            }
            OpCode::GETUPVAL => {
                let f = self.get_fn().unwrap();
                let uvs = &as_fun(f).upvals;
                let uv = uvs[i.b() as usize];
                self.set_stack_slot(as_ci(self.ci).base + i.a(), as_uv(uv).v);
            }
            OpCode::CLOSURE => {
                let mut ls = LocalScope::new();
                let f = self.gc.new_fun(false);
                as_fun(f).f = self.get_fn_tpl(i.bx()) as *const c_void;
                ls.reg(f);

                let uv_desc = &as_fun(f).tpl().upvals;
                uv_desc.iter().for_each(|uvd| {
                    if uvd.in_stack {
                        let uv = self.find_upvals(uvd);
                        as_obj(uv).inc();
                        as_fun(f).upvals.push(uv);
                    } else {
                        let cf = self.get_fn().unwrap();
                        let uvs = &as_fun(cf).upvals;
                        let uv = *uvs.get(uvd.idx as usize).unwrap();
                        let uv = self.gc.new_upval(as_uv(uv).v, false);
                        as_fun(f).upvals.push(uv);
                    }
                });

                self.set_stack_slot(as_ci(self.ci).base + i.a(), as_obj_ptr(f));
            }
            OpCode::MOVE => {
                let base = as_ci(self.ci).base;
                let v = self.get_stack_item(base + i.b());
                self.set_stack_slot(base + i.a(), v);
            }
            OpCode::TEST => {
                let base = as_ci(self.ci).base;
                let v = self.get_stack_item(base + i.a());
                let b = as_obj(v).t_bool();
                let com = if as_obj(b).eqs_true() { 1 } else { 0 };
                let eq = com == i.c();
                if eq {
                    as_ci(self.ci).pc += 1;
                }
            }
            OpCode::TESTSET => {
                let base = as_ci(self.ci).base;
                let b = self.get_stack_item(base + i.b());
                let tb = as_obj(b).t_bool();
                let com = if as_obj(tb).eqs_true() { 1 } else { 0 };
                let eq = com == i.c();
                if eq {
                    as_ci(self.ci).pc += 1;
                } else {
                    self.set_stack_slot(base + i.a(), b);
                }
            }
            OpCode::JMP => {
                let pc = as_ci(self.ci).pc as i32;
                as_ci(self.ci).pc = (pc + i.sbx()) as u32;
            }
            OpCode::LOADBOO => {
                let base = as_ci(self.ci).base;
                let b = if i.b() == 1 {
                    self.gc.js_true()
                } else {
                    self.gc.js_false()
                };
                self.set_stack_slot(base + i.a(), as_obj_ptr(b));
                if i.c() == 1 {
                    as_ci(self.ci).pc += 1;
                }
            }
            OpCode::EQ | OpCode::LT | OpCode::LE => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;
                let b = self.get_stack_item(base + i.b());
                let (b, is_new) = self.x_rk(base, i.b());
                if is_new {
                    ls.reg(b);
                }
                let (c, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(c);
                }
                let com = match op {
                    OpCode::EQ => GcObj::eq(b, c),
                    OpCode::LT => GcObj::lt(b, c),
                    OpCode::LE => GcObj::le(b, c),
                    _ => unimplemented!(),
                };
                let com = if com { 1 } else { 0 };
                if com != i.a() {
                    as_ci(self.ci).pc += 1;
                }
            }
            OpCode::ADD | OpCode::SUB | OpCode::MUL | OpCode::DIV | OpCode::MOD => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;

                let (a, is_new) = self.x_rk(base, i.b());
                if is_new {
                    ls.reg(a);
                }
                let a = as_obj(a).t_num();
                ls.reg(a);

                let (b, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(b);
                }
                let b = as_obj(b).t_num();
                ls.reg(b);

                let v = match op {
                    OpCode::ADD => as_num(a).add(as_num(b)),
                    OpCode::SUB => as_num(a).sub(as_num(b)),
                    OpCode::MUL => as_num(a).mul(as_num(b)),
                    OpCode::DIV => as_num(a).div(as_num(b)),
                    OpCode::MOD => as_num(a).modulo(as_num(b)),
                    _ => panic!(),
                };
                ls.reg(v);
                self.set_stack_slot(as_ci(self.ci).base + i.a(), as_obj_ptr(v));
            }
            OpCode::NEWTABLE => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;
                let v = self.gc.new_dict(false);
                ls.reg(v);
                self.set_stack_slot(base + i.a(), as_obj_ptr(v));
            }
            OpCode::SETTABLE => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;
                let tb = self.get_stack_item(base + i.a());
                if !as_obj(tb).check_coercible() {
                    panic!("TypeError")
                }
                let tb = as_dict(tb);
                let (k, is_new) = self.x_rk(base, i.b());
                if is_new {
                    ls.reg(k);
                }
                let (v, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(v);
                }
                tb.set(k, v);
            }
            OpCode::GETTABLE => {
                let mut ls = LocalScope::new();
                let base = as_ci(self.ci).base;
                let tb = self.get_stack_item(base + i.b());
                if !as_obj(tb).check_coercible() {
                    panic!("TypeError")
                }
                // TODO:: dispatch to various prop resolve handler(number, string, array)
                let tb = as_dict(tb);
                let (k, is_new) = self.x_rk(base, i.c());
                if is_new {
                    ls.reg(k);
                }
                let v = tb.get(k);
                self.set_stack_slot(base + i.a(), v);

                // set this for calling function on object
                as_ci(self.ci).set_this(as_obj_ptr(tb));
            }
            OpCode::RETURN => {
                let b = i.b();
                let mut ret_num = b - 1;
                if as_ci(self.ci).is_new {
                    ret_num = 1;
                    let ret = as_ci(self.ci).this;
                    self.set_stack_slot(as_ci(self.ci).fun, ret);
                } else if ret_num == 1 {
                    let ret = self.get_stack_item(as_ci(self.ci).base + i.a());
                    self.set_stack_slot(as_ci(self.ci).fun, ret);
                }
                self.post_call(ret_num);
            }
            OpCode::CALL => {
                let fi = as_ci(self.ci).base + i.a();
                let fp = self.get_stack_item(fi);
                assert_eq!(
                    as_obj(fp).kind,
                    GcObjKind::Function,
                    "Uncaught TypeError: not a function"
                );
                let f = as_fun(fp);

                let ci = CallInfo::new();
                as_ci(ci).fun = fi;
                as_ci(ci).base = fi + i.b();

                as_ci(ci).prev = self.ci;
                let cci = self.ci;
                as_ci(cci).next = ci;
                as_ci(ci).set_this(as_ci(cci).this);
                self.ci = ci;

                if f.is_native {
                    as_ci(ci).is_native = true;
                    unsafe {
                        let f = std::mem::transmute::<*const c_void, NativeFn>(f.f);
                        f(self)
                    }
                    self.post_call(i.c() - 1);
                } else {
                    let mut ls = LocalScope::new();
                    let n_args = i.b() - 1;
                    let r_arg = as_ci(self.ci).fun + 1;
                    let t_arg = as_ci(self.ci).base;
                    for i in 0..n_args {
                        let mut v = self.get_stack_item(r_arg + i);
                        if as_obj(v).pass_by_value() {
                            v = as_obj(v).x_pass_by_value();
                            ls.reg(v);
                        }
                        self.set_stack_slot(t_arg + i, v);
                    }
                    self.exec();
                }
            }
            OpCode::THIS => {
                let base = as_ci(self.ci).base;
                let this = as_ci(self.ci).this;
                self.set_stack_slot(base + i.a(), this);
            }
            OpCode::NEW => {
                let fi = as_ci(self.ci).base + i.a();
                let fp = self.get_stack_item(fi);
                assert_eq!(
                    as_obj(fp).kind,
                    GcObjKind::Function,
                    "Uncaught TypeError: not a function"
                );
                let f = as_fun(fp);

                let ci = CallInfo::new();
                as_ci(ci).fun = fi;
                as_ci(ci).base = fi + i.b();

                as_ci(ci).prev = self.ci;
                let cci = self.ci;
                as_ci(cci).next = ci;
                self.ci = ci;

                // we'll implement this later
                if f.is_native {
                    unimplemented!()
                }

                let mut ls = LocalScope::new();
                let this = self.gc.new_dict(false);
                ls.reg(this);

                let n_args = i.b() - 1;
                let r_arg = as_ci(self.ci).fun + 1;
                let t_arg = as_ci(self.ci).base;
                for i in 0..n_args {
                    let mut v = self.get_stack_item(r_arg + i);
                    if as_obj(v).pass_by_value() {
                        v = as_obj(v).x_pass_by_value();
                        ls.reg(v);
                    }
                    self.set_stack_slot(t_arg + i, v);
                }

                as_ci(self.ci).set_this(as_obj_ptr(this));
                as_ci(self.ci).is_new = true;
                self.exec();
            }
            // TODO::
            _ => (),
        }
        Ok(())
    }
}

impl Vm {
    pub fn register_native_fn(&mut self, name: &str, nf: NativeFn) {
        let f = self.gc.new_fun_native(nf, false);
        as_dict(self.env).insert(name, as_obj_ptr(f));
    }
}

#[cfg(test)]
mod exec_tests {
    use super::*;
    use crate::asm::codegen::*;

    #[test]
    fn exec_init_test() {
        let chk = Codegen::gen(
            "var a = 1
    print(a)
    assert(1, a)
    print(native_fn())
    ",
        );
        let mut vm = Vm::new(chk, 1024);

        vm.register_native_fn("assert", |vm: VmPtr| {
            let args = as_vm(vm).get_args();
            assert_eq!(args.len(), 2);
            let a = args[0];
            let b = args[1];
            assert_eq!(as_num(a).d, as_num(b).d);
            println!("assert ok");
        });

        vm.register_native_fn("native_fn", |vm: VmPtr| {
            let mut ls = LocalScope::new();
            let ret = as_vm(vm).gc.new_str(false);
            ls.reg(ret);
            as_str(ret).d.push_str("return from native call");
            as_vm(vm).set_return(as_obj_ptr(ret));
        });

        vm.exec();
    }

    fn new_vm(chk: Chunk) -> Box<Vm> {
        let mut vm = Vm::new(chk, 1 << 20);

        vm.register_native_fn("assert_str_eq", |vm: VmPtr| {
            let args = as_vm(vm).get_args();
            assert_eq!(args.len(), 2);
            let a = args[0];
            let b = args[1];
            assert_eq!(as_str(a).d, as_str(b).d);
        });

        vm.register_native_fn("assert_num_eq", |vm: VmPtr| {
            let args = as_vm(vm).get_args();
            assert_eq!(args.len(), 2);
            let a = args[0];
            let b = args[1];
            assert_eq!(as_num(a).d, as_num(b).d);
        });

        vm
    }

```
