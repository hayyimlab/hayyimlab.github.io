---
title: The context sensitivity of C and Rust's grammar
author: "Donghyun"
tags: [cfg, context-free, context-sensitive, context-free language, regular language, programming language, parsing, parser, raw string literal, pumping lemma, ambiguity]
toc: true
---

## Context-free grammar

[Context-free grammar(CFG)](https://en.wikipedia.org/wiki/Context-free_grammar)는 프로그래밍 언어의 코드를 parsing 하는 데에 유용한 이론적 도구로 쓰입니다. 일례로, parsing 도구 중 하나인 [Yacc](http://en.wikipedia.org/wiki/Yacc)는 CFG용 parser를 생성합니다. 그러나 실제로는 다수의 프로그래밍 언어가 context-free 하지 않은 문법을 가집니다.

## C

C는 널리 사용되는 언어 중 하나이며 *거의* context-free 한 문법을 가져 위 내용을 설명하기 좋은 예입니다.

CFG는 형식 언어 및 프로그래밍 언어와 관련하여 다양한 방식으로 정의됩니다. 본 글에서는 명명법에 대해 깊이 파고들지는 않습니다-이에 관한 내용이 궁금하다면 [관련 토론](https://groups.google.com/g/comp.lang.misc/c/MCZmQv56--Q?pli=1#39cd8a1804f11cca)을 참고하세요. 본 글에서 C의 문법이 CFG가 아니라고 하는 것은, Yacc[^1]에 주어진 문법이 다른 곳에서 오는 몇몇 context information을 참조하지 않고는 C를 올바르게 parsing 하기에 충분하지 않다는 것을 의미합니다. 몇 가지 예를 들어보겠습니다.

```c
{
  T (x);
  ...
}
```

생소할 수도 있지만, `T`가 type이면 상기 C 코드는 type `T`의 `x`에 대한 유효한 선언입니다. 그러나 `T`가 알려진 type이 아니면 argument `x`를 가진 function `T`에 대한 호출이 됩니다. C parser는 `T`가 이전에 `typedef`에 의해 정의되었는지 여부를 알지 못한 채 어떻게 parsing 할 방법을 알 수 있을까요?

혹자는 "하지만 누가 이런 코드를 작성하나요?"라고 할 수도 있을 겁니다. 좋아요, 그렇다면 좀 더 표준적인 코드를 보죠:

```c
{
  T * x;
  ...
}
```

이것은 `T`에 대한 pointer로 `x`를 선언한 것일까요, 아니면 variable `T`와 `x`의 곱셈일까요? 메모리에 `typedef`로 정의된 type에 대한 table이 없으면 알 수 없습니다-이는 *context-sensitive information*입니다.

다른 예가 있습니다:

```c
func((T) * x);
```

`T`가 type이면 `x`를 역참조한 결과가 `T`로 형변환되어 `func`에 전달됩니다. `T`가 type이 아니면 `T`와 `x`의 곱셈이 `func`에 전달됩니다.

위 모든 예에서, 문제를 유발할 수 있는 부분에 도달하기 전에 코드에서 context-sensitive information을 수집하지 않으면 parser는 정상적으로 작동하지 못합니다. 이 문제는 compilation/C 커뮤니티에서 "typedef-name: identifier" 문제로 불립니다. K&R2[^2]도 부록에서 C의 문법을 제시할 때 이를 언급합니다:

> With one further change, namely deleting the production *typedef-name: identifier* and making *typedef-name* a terminal symbol, this grammar is acceptable to the YACC parser-generator.

다행히도 이 문제는 간단히 해결할 수 있습니다. Parsing이 진행되는 동안 `typedef`로 정의된 type의 symbol table을 유지하기만 하면 됩니다. Lexer에서 새 identifier가 인식될 때마다 이 identifier가 정의된 type인지 확인하고 올바른 token을 parser에 반환합니다. Parser는 identifier와 정의된 type이라는 두 가지 terminal을 가지며, 이제 남은 것은 `typedef` 문의 parsing이 완료될 때마다 symbol table을 업데이트하는 것입니다. 이것이 어떻게 작동하는지 더 잘 이해하기 위해 c2c의 코드에서 C parser와 lexer 관련 부분을 살펴봅시다. 다음은 Lex 파일의 일부입니다:

```
identifier ([a-zA-Z_][0-9a-zA-Z_]*)

<INITIAL,C>{identifier} 
  { 
    GetCoord(&yylval.tok);  
    yylval.n = MakeIdCoord(UniqueString(yytext), 
                           yylval.tok);
    if (IsAType(yylval.n->u.id.text))
      RETURN_TOKEN(TYPEDEFname);
    else 
      RETURN_TOKEN(IDENTIFIER); 
  }
```

여기서 Lex의 syntax를 자세히 설명하지 않더라도, 이것이 기본적으로 말하는 것은 identifier가 발견될 때마다 해당 identifier가 type인지 확인한다는 것입니다. Type이면 `TYPEDEFname` token이 반환되고, type이 아니면 `IDENTIFIER`가 반환됩니다. Yacc 문법에서 이 둘은 별도의 terminal입니다.

## Rust

Rust의 lexical 문법은 context-free 하지 않습니다. [Raw string literal](https://doc.rust-lang.org/stable/reference/tokens.html#raw-string-literals)이 그 원인입니다. Raw string literal은 `r`에 이은 N개의 hash(N은 0일 수 있음), 따옴표, 임의의 문자들, 따옴표 그리고 N개의 hash로 구성됩니다. 중요한 것은 첫 번째 따옴표 쌍 안에 다른 따옴표가 들어오면 그 뒤에 N개의 hash가 연속적으로 올 수 없다는 것입니다. 가령, `r###""###"###`은 유효하지 않습니다.

아래 문법을 봅시다:

```
R -> 'r' S
S -> '"' B '"'
S -> '#' S '#'
B -> . B
B -> ε
```

여기서 `.`는 임의의 문자를 나타내고 `ε`는 빈 문자열을 나타냅니다. 문자열 `r#""#"#`을 생각해 보면, 이 문자열은 유효한 raw string literal은 아니지만 다음과 같은 유도 과정을 통해 상기 문법에서는 허용됩니다:

```
R : #""#"#
S : ""#"
S : "#
B : #
B : ε
```

(여기서 `T : U`는 규칙 `T`가 적용됨을 의미하고 `U`는 문자열의 나머지 부분입니다.) 근본적으로 raw string literal이 context-sensitive 하므로 CFG로 표현 시에 어려움이 발생하는 것이며, 특히 필요한 context는 hash의 수 입니다.

Rust의 string literal이 context-free 하지 않음을 증명하기 위해 context-free language가 regular language와의 교집합에 대해 닫혀 있다는 사실과 [context-free language에 대한 pumping lemma](https://en.wikipedia.org/wiki/Pumping_lemma_for_context-free_languages)를 사용할 것입니다.

Regular language `R = r#+""#*"#+`를 생각해 봅시다. Rust의 raw string literal이 context-free 하면, raw string literal과 `R`의 교집합인 `R'` 역시 context-free 해야 합니다. 따라서 raw string literal이 context-free 하지 않음을 증명하려면 `R'`이 context-free 하지 않음을 증명하면 됩니다.

`R'`은 `{r#^n""#^m"#^n | m < n}`입니다.

`R'`이 context-free 하다고 가정합시다. 그러면 `R'`은 pumping lemma가 적용되는 pumping length `p > 0`를 가집니다. `R'`에서 다음 문자열 `s`를 생각해 봅시다:

`` r#^p""#^{p-1}"#^p ``

e.g. for `p = 2`: `s = r##""#"##`

아래 조건을 만족시키도록 `s = uvwxy`와 같이 분할할 수 있습니다.

- `|vx| ≥ 1`
- `|vwx| ≤ p`
- `uv^iwx^iy ∈ R'` for all `i ≥ 0`

`R'`의 모든 문자열에서 `"`와 `r`의 수는 고정돼 있으므로 `v`와 `x` 모두 `"` 또는 `r`을 포함할 수 없습니다. 따라서 `v`와 `x`는 hash만 포함합니다. 결과적으로, 세 개의 hash sequence 중 `v`와 `x`를 합친 것은 그중 두 개만 pumping 할 수 있습니다. 만약 중앙 hash sequence를 선택하면 pumping 시 외부 sequence 중 하나가 증가하지 않아 외부 sequence 간의 불균형이 발생합니다. 따라서 두 외부 hash sequence를 모두 pumping 해야 합니다. 그러나 이 두 hash sequence 사이에는 `p + 2`개의 문자가 존재하며, `|vwx|`는 `p`보다 작거나 같아야 합니다. 여기서 모순이 발생하여 `R'`이 context-free 하지 않음이 도출됩니다.

`R'`가 context-free 하지 않으므로 Rust의 raw string literal 또한 context-free 하지 않습니다.

## References

- [https://eli.thegreenplace.net/2007/11/24/the-context-sensitivity-of-cs-grammar](https://eli.thegreenplace.net/2007/11/24/the-context-sensitivity-of-cs-grammar)
- [https://github.com/rust-lang/rust/blob/cb8ab33ed29544973da866bdc3eff509b3c3e789/src/grammar/raw-string-literal-ambiguity.md](https://github.com/rust-lang/rust/blob/cb8ab33ed29544973da866bdc3eff509b3c3e789/src/grammar/raw-string-literal-ambiguity.md)
- [https://matklad.github.io/2018/06/06/modern-parser-generator.html](https://matklad.github.io/2018/06/06/modern-parser-generator.html)



*수정 및 보충 사항은 [dominic2009@snu.ac.kr](mailto:dominic2009@snu.ac.kr)로 제보 바랍니다.*

---

[^1]: Yacc는 CFG만 허용합니다.
[^2]: "The ANSI C programming language, 2nd edition" by Kernighan and Ritchie
