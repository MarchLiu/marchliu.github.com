---
layout: post
title: "Notes of Write Scheme 48 Hours"
description: "这是一份学习笔记，《48 Hours》通过开发一个 Scheme 解释器介绍了 Haskell 语言"
category: notes
tags: [learn, notes, haskell]
---


## 原子词素解析器

~~~

parseAtom :: Parser LispVal
parseAtom = do first <- letter <|> symbol -- 字母或操作符开头
               rest <- many (letter <|> digit <|> symbol) -- 一到多个
               let atom = [first] ++ rest -- 迭加
               return $ case atom of -- 根据词素解析生成返回的解析值
                          "#t" -> Bool True 
                          "#f" -> Bool False
                          otherwise -> Atom atom

~~~
{: .language-haskell}

### 练习 2.3.2 
(Number . read) 如果“.”两边没有空格会出错，应该是错把 Number.read 当 作一个类型了。
看了这个才知道我对 Monad 还是玩的不熟，意识不够：

~~~

parseNumber :: Parser LispVal
parseNumber = many1 digit >>= (return . Number . read)

~~~
{: .language-haskell}

### 练习 2.3.3 

这个逃逸字符处理的思路我想到了，但是还是没有意识到 do 模式下 many 和 return 可以这样简洁漂亮的组合：

~~~
escapedChars :: Parser Char
escapedChars = do char '\\' 
                  x <- oneOf "\\\"nrt" 
                  return $ case x of 
                    '\\' -> x
                    '"'  -> x
                    'n'  -> '\n'
                    'r'  -> '\r'
                    't'  -> '\t'
~~~
{: .language-haskell}

附扩展后的 parseString ，这个倒没大变化：

~~~
parseString :: Parser LispVal
parseString = do char '"'
                 x <- many $ escapedChars <|> noneOf "\"\\"
                 char '"'
                 return $ String x
~~~
{: .language-haskell}

### 练习 2.3.4 

基本上进制不同的解析是一个力气活。我原本想用 do return case 的方式，但 是后来看还是答案中的每种进制分别解析比较合理，因为逻辑上要区分一些特殊情况。

这其中需要注意的知识点就是 Numeric 模块中的 readXXX 函数都是返回二元组， 要像如下代码这样取第一个元素出来才是我们需要的答案。

~~~
oct2dig x = fst $ readOct x !! 0
hex2dig x = fst $ readHex x !! 0
bin2dig  = bin2dig' 0
bin2dig' digint "" = digint
bin2dig' digint (x:xs) = let old = 2 * digint + (if x == '0' then 0 else 1) in
                         bin2dig' old xs
~~~
{: .language-haskell}

### 练习 2.3.5 
这里在函数内部嵌套 do 的技巧值得学习

~~~
parseCharacter :: Parser LispVal
parseCharacter = do
  try $ string "#\\"
  value <- try (string "newline" <|> string "space")
           <|> do {x <- anyChar; notFollowedBy alphaNum; return [x]}
  return $ Character $ case value of
                         "space" -> ' '
                         "newline" -> '\n'
                         otherwise -> (value !! 0)
~~~
{: .language-haskell}

这里开始更多的使用 try ，这个函数我还不熟悉。

~~~
parseExpr :: Parser LispVal
parseExpr = parseAtom
            <|> parseString
            <|> try parseNumber
            <|> try parseBool
            <|> try parseCharacter
~~~

### 练习 2.3.7 
原文这里似乎有错，我最终用的是 parseNumber 而不是 parseDecimal，这是考 虑到应该允许输入不同进制的复数。

~~~
parseComplex :: Parser LispVal
parseComplex = do x <- (try parseFloat <|> parseDecimal)
                  char '+' 
                  y <- (try parseFloat <|> parseDecimal)
                  char 'i' 
                  return $ Complex (toDouble x :+ toDouble y)
~~~

## 3.4 Recursive Parsers 
CLOSED: 2010-11-23 二 01:00
递归解释器这里令人赞叹的利用了 monad then ： CLOSED: 2010-11-23 二 01:00

~~~
parseDottedList :: Parser LispVal
parseDottedList = do
    head <- endBy parseExpr spaces
    tail <- char '.' >> spaces >> parseExpr
    return $ DottedList head tail
~~~
{: .language-haskell}

有前趋状态（如某些数据会引用前缀，类似数组则会迭加元素）的时候，需要用 try 。可以确保出错时回溯到出错前的状态。

在加入了练习补充内容的代码中增加 list 解析过程时，会有编译错误，需要 把 list 的解析器单独取出来。

~~~
parseLst :: Parser LispVal
parseLst = do char '('
              x <- (try parseList) <|> parseDottedList
              char ')'
              return x
~~~
{: .language-haskell}
然后 parseExpr 就成为这样：

~~~
parseExpr :: Parser LispVal
parseExpr = parseAtom
            <|> parseString
            <|> try parseNumber
            <|> try parseRatio
            <|> try parseFloat
            <|> try parseComplex
            <|> try parseBool
            <|> try parseCharacter
            <|> parseQuoted
            <|> parseLst
~~~
{: .language-haskell}

在本节的几个解析器实现中，递归调用 parseExpr ，实现了整个解析器对递归语 法结构的解析。

### 练习 2.4.3 

在这里 wikibooks 给出了两组不同的实现，一个是实现一种 AnyList。此实现有 一些错误，首先是没有给出 Nil 的实现，我自己尝试做了一个：

~~~
data LispVal = Atom String
             | List [LispVal]
             | DottedList [LispVal] LispVal
             | Number Integer
             | Float Double
             | String String
             | Bool Bool
             | Character Char
             | Ratio Rational
             | Complex (Complex Double)
             | Vector (Array Int LispVal)
             | Nil ()
~~~
{: .language-haskell}

然后 optionalSpaces 也不存在，实际上应该就是 spaces。

~~~
parseAnyList :: Parser LispVal
parseAnyList = do
  char '('
  spaces
  head <- sepEndBy parseExpr spaces
  tail <- (char '.' >> spaces >> parseExpr) <|> return (Nil ())
  spaces
  char ')'
  return $case tail of
            (Nil ()) -> List head
            otherwise -> DottedList head tail
~~~
{: .language-haskell}

该页上还给出一种方法是扩展 parseList ，此方法不使用 Nil。

~~~
-- parseList' :: Parser LispVal
parseList' head = do char '.' >> spaces1
                     tail <- parseExpr
                     spaces >> char ')'
                     return $ DottedList head tail

parseList :: Parser LispVal
parseList = do char '(' >> spaces
               head <- parseExpr `sepEndBy` spaces1
               (parseList' head)<|> (spaces >> char ')' >> (return $ List head))
~~~
{: .language-haskell}

此例中仍然遇到在有 choice 运算符（<|>） 的情况下无法嵌套 do，于是定义了一个辅助函数。

## Evalution, Part 1

### 4.2 Beginnings of an evaluator: Primitives 
因为 Lisp 族系代码即数据，所以 eval LispVal 变量就是返回 LispVal本身。

这里使用 @ 运算符，绑定进行局部匹配的变量：

~~~
eval :: LispVal -> LispVal
eval val@(String _) = val
eval val@(Number _) = val
eval val@(Bool _) = val
eval (List [Atom "quote", val]) = val
~~~
{: .language-haskell}

加入 eval 后的主函数再次体现了 monad 绑定与 . 运算符的组合技巧。

~~~
main :: IO ()
main = getArgs >>= putStrLn . show . eval . readExpr . (!! 0)
~~~
{: .language-haskell}

这个函数中的 ($args) 巧妙使用了 $ 运算符的右结合

~~~
apply :: String -> [LispVal] -> LispVal
apply func args = maybe (Bool False) ($ args) $ lookup func primitives
~~~
{: .language-haskell}

## Intermezzo: Error Checking & Exceptions

错误处理一章中展示了 Either 的运用。包括 throwError 和 catchError。

在我们的项目中，用 trapError 方法将 catchError 包装起来。

~~~
trapError action = catchError action (return . show)
~~~
{: .language-haskell}

对于正确的执行结果，我们用 extractValue 析出结果值：

~~~
extractValue :: ThrowsError a -> a
extractValue (Right val) = val
~~~
{: .language-haskell}

eval 中要多处理出错的情况

~~~
eval :: LispVal -> ThrowsError LispVal
eval val@(String _) = return val
eval val@(Number _) = return val
eval val@(Bool _) = return val
eval (List [Atom "quote", val]) = return val
eval (List (Atom func : args)) = mapM eval args >>= apply func
eval badForm = throwError $ BadSpecialForm "Unrecognized special form" badForm
~~~
{: .language-haskell}

函数 primitives 只需要修改返回值声明

~~~
primitives :: [(String, [LispVal] -> ThrowsError LispVal)]
~~~
{: .language-haskell}

...

因为我们不需要修改它的每一种实现，只要修改 unaryOp 和 numericBinop 函数即可：

~~~
numericBinop :: (Integer -> Integer -> Integer) -> [LispVal] -> ThrowsError LispVal
numericBinop op singleVal@[_] = throwError $ NumArgs 2 singleVal
numericBinop op params = mapM unpackNum params >>= return . Number . foldl1 op 

unaryOp :: (LispVal -> LispVal) -> [LispVal] -> ThrowsError LispVal
unaryOp f [v] = return $ f v
~~~
{: .language-haskell}

unpackNum 中，我们对无效值作了异常抛出。如果想要严格类型约束，可以将针对 String 和 List 的实现注释掉。

~~~
unpackNum :: LispVal -> ThrowsError Integer
unpackNum (Number n) = return n
unpackNum (String n) = let parsed = reads n in
                       if null parsed
                          then throwError $ TypeMismatch "number" $ String n
                          else return $ fst ( parsed !! 0)
unpackNum (List [n]) = unpackNum n
unpackNum notNum = throwError $ TypeMismatch "number" notNum
~~~
{: .language-haskell}

主函数 main 的实现中，通过 trapError 捕获了可能出现的错误

~~~
main :: IO ()
main = do
  args <- getArgs
  evaled <- return $ liftM show $ readExpr (args !! 0) >>= eval
  putStrLn $ extractValue $ trapError evaled
~~~
{: .language-haskell}

这里对 apply 也做了扩展

~~~
apply :: String -> [LispVal] -> ThrowsError LispVal
apply func args = maybe (throwError $ NotFunction "Unrecognized primitives function args" func) 
                  ($ args) 
                  (lookup func primitives)
~~~
{: .language-haskell}

这里利用 Either 进行异常和错误处理的方法值得关注。

## Evalution, Part 2

### Additional Primitives: Partial Application 

类似 unpackNum ，现在我们也有了 unpackStr 和 unpack Num。 notString 和 notBool 的错误值匹配也是一个技巧。

~~~
unpackStr :: LispVal -> ThrowsError String
unpackStr (String s) = return s
unpackStr (Number s) = return $ show s
unpackStr (Bool s) = return $ show s
unpackStr notString = throwError $ TypeMismatch "string" notString

unpackBool :: LispVal -> ThrowsError Bool
unpackBool (Bool b) = return b
unpackBool notBool = throwError $ TypeMismatch "boolean" notBool
~~~
{: .language-haskell}

二元逻辑判断的实现。

~~~
boolBinop :: (LispVal -> ThrowsError a) -> (a -> a -> Bool) -> [LispVal] -> ThrowsError LispVal
boolBinop unpacker op args = if length args /= 2
                             then throwError $ NumArgs 2 args
                             else do left <- unpacker $args !! 0
                                     right <- unpacker $args !! 1
                                     return $ Bool $ left `op` right

numBoolBinop = boolBinop unpackNum
strBoolBinop = boolBinop unpackStr
boolBoolBinop = boolBinop unpackBool
~~~
{: .language-haskell}

书中给出的代码有些错误，现在完整的指令表是：

~~~
primitives :: [(String, [LispVal] -> ThrowsError LispVal)]
primitives = [("+", numericBinop (+)),
              ("-", numericBinop (-)),
              ("*", numericBinop (*)),
              ("/", numericBinop div),
              ("=", numBoolBinop (==)),
              ("<", numBoolBinop (<)),
              (">", numBoolBinop (>)),
              ("/=", numBoolBinop (/=)),
              (">=", numBoolBinop (>=)),
              ("<=", numBoolBinop (<=)),
              ("&&", boolBoolBinop (&&)),
              ("||", boolBoolBinop (||)),
              ("string=?", strBoolBinop (==)),
              ("string>?", strBoolBinop (>)),
              ("string<?", strBoolBinop (<)),
              ("string<=?", strBoolBinop (<=)),
              ("string>=?", strBoolBinop (>=)),
              ("mod", numericBinop mod),
              ("quotient", numericBinop quot),
              ("remainder", numericBinop rem),
              ("symbol?", unaryOp symbolp),
              ("string?", unaryOp stringp),
              ("number?", unaryOp numberp),
              ("bool?", unaryOp boolp),
              ("list?", unaryOp listp)]
~~~

### Pattern Matching 2 
由于 if 语句的实现没有走 primitives ，而是直接取函数名，这里需要将它的 eval 放在前面：

~~~
eval :: LispVal -> ThrowsError LispVal
eval val@(String _) = return val
eval val@(Number _) = return val
eval val@(Bool _) = return val
eval (List [Atom "quote", val]) = return val
eval (List [Atom "if", pred, conseq, alt]) =
    do result <- eval pred
       case result of
         Bool False -> eval alt
         otherwise -> eval conseq
eval (List (Atom func : args)) = mapM eval args >>= apply func
eval badForm = throwError $ BadSpecialForm "Unrecognized special form" badForm
~~~
{: .language-haskell}

### List Primitives: car, cdr, and cons 

对于 car 来说，实现代码与匹配规则一一对应：

~~~
(car (a b c)) = a
(car (a)) = a
(car (a b . c)) = a
(car a) = error (not a list)
(car a b) = error (car takes only one argument)
~~~
{: .language-haskell}
实现为：

~~~
car :: [LispVal] -> ThrowsError LispVal
car [List (x : xs)] = return x
car [DottedList (x : xs) _] = return x
car [badArg] = throwError $ TypeMismatch "pair" badArg
car badArgList = throwError $ NumArgs 1 badArgList
~~~
{: .language-haskell}

同理， cdr 规则

~~~
(cdr (a b c)) = (b c)
(cdr (a b)) = (b)
(cdr (a)) = NIL
(cdr (a b . c)) = (b . c)
(cdr (a . b)) = b
(cdr a) = error (not list)
(cdr a b) = error (too many args)
~~~
{: .language-haskell}

实现为：

~~~
cdr :: [LispVal] -> ThrowError LispVal
cdr [List (x:xs)] = return $ List xs
cdr [DottedList (_ :xs) x] = return $ DottedList xs x
cdr [DottedList [xs] x] = return x
cdr [badArg] = throwError $ TypeMismatch "pari" badArg
cdr badArgList = throwError $ NumArgs 1 badArgList
~~~
{: .language-haskell}

同理添加 cons 和 eqv ，在 primitives 加入对应的 KV 对。

### Equal? and Weak Typing: Heterogenous Lists 

这里利用了 GHC 的扩展： Existential Types 。使用时需要调用 GHC 的 -fglasgow-exts 选项。

原文这两行似乎写反了。

~~~
cdr [DottedList [xs] x] = return x
cdr [DottedList (_ :xs) x] = return $ DottedList xs x
~~~
{: .language-haskell}

这部分展示了一些使用扩展编译选项的类型推导技术，如 forall。

~~~
data Unpacker = forall a . Eq a => AnyUnpacker (LispVal -> ThrowsError a)
~~~
{: .language-haskell}

forall 的自动类型推导使得弱类型容器成为可能，进一步允许弱类型比较， 最终落实这个运算的是强和弱判等：

~~~
eqv :: [LispVal] -> ThrowsError LispVal
eqv [(Bool arg1), (Bool arg2)] = return $ Bool $ arg1 == arg2
eqv [(Number arg1), (Number arg2)] = return $ Bool $ arg1 == arg2
eqv [(String arg1), (String arg2)] = return $ Bool $ arg1 == arg2
eqv [(Atom arg1), (Atom arg2)] = return $ Bool $ arg1 == arg2
eqv [(DottedList xs x), (DottedList ys y)] = eqv [List $ xs ++ [x], List $ ys ++ [y]]
eqv [(List arg1), (List arg2)] = return $ Bool $ (length arg1 == length arg2) &&
                                 (and $ map eqvPair $ zip arg1 arg2)
                                 where eqvPair (x1, x2) = case eqv [x1, x2] of
                                                            Left err -> False
                                                            Right (Bool val) -> val
eqv [_, _] = return $ Bool False
eqv badArgList = throwError $ NumArgs 2 badArgList

equal :: [LispVal] -> ThrowsError LispVal
equal [arg1, arg2] = do
  primitiveEqual <- liftM or $ mapM (unpackEquals arg1 arg2)
                    [AnyUnpacker unpackNum, AnyUnpacker unpackStr, AnyUnpacker unpackBool]
  eqvEquals <- eqv [arg1, arg2]
  return $ Bool $ (primitiveEqual || let (Bool x) = eqvEquals in x)

equal badArgList = throwError $ NumArgs 2 badArgList
~~~
{: .language-haskell}

### 练习 6.4.1 
这个没太大技术含量，我自己就搞定了：

~~~
eval (List [Atom "if", pred, conseq, alt]) =
    do result <- eval pred
       case result of
         Bool True -> eval conseq
         Bool False -> eval alt
         otherwise -> throwError $ TypeMismatch "bool" pred
~~~
{: .language-haskell}

### 练习 6.4.2 

对 List 进行强和弱类型比较，这里比我想像的复杂，作者先实现了一个 List 比较函数。

~~~
eqvList :: ([LispVal] -> ThrowsError LispVal) -> [LispVal] -> ThrowsError LispVal
eqvList eqvFunc [(List arg1), (List arg2)] = return $ Bool $ (length arg1 == length arg2) &&
                                             (all eqvPair $ zip arg1 arg2)
    where eqvPair (x1, x2) = case eqvFunc [x1, x2] of
                               Left err -> False
                               Right (Bool val) -> val
~~~
{: .language-haskell}

然后用它重写了 eqv 和 equal 的 List 实现：

~~~
eqv [l1@(List arg1), l2@(List arg2)] = eqvList eqv [l1, l2]
eqv [(DottedList xs x), (DottedList ys y)] = eqv [List $ xs ++ [x], List $ ys ++ [y]]
...
equal [l1@(List arg1), l2@(List arg2)] = eqvList equal [l1, l2]
equal [(DottedList xs x), (DottedList ys y)] = equal [List $ xs++[x], List $ ys++[y]]
~~~
{: .language-haskell}

因为haskell的函数模式匹配是按照书写顺序尝试的，所以要注意 函数的顺序。

### 练习 6.4.3 

作者给出了两种不同的 cond 实现，这里我采用了第一种：

~~~~
eval (List ((Atom "cond"):cs)) = do
  b <- (liftM (take 1 . dropWhile f) $ mapM condClause cs) >>= cdr
  car [b] >>= eval
      where condClause (List [Atom "else", b]) = return $ List [Bool True, b]
            condClause (List [p,b]) = do q <- eval p
                                         case q of
                                           Bool _ -> return $ List [q,b]
                                           _ -> throwError $ TypeMismatch "bool" q
            condClause v = throwError $ TypeMismatch "(pred body)" v
            f = \(List [p,b]) -> case p of
                                   (Bool False) -> True
                                   _ -> False
~~~~
{: .language-haskell}

根据这个代码，我做出了 case 的实现：

~~~
eval (List ((Atom "case"):cs)) = do
  b <- (liftM (take 1 . dropWhile f ) $ mapM condClause (tail cs)) >>= cdr
  car [b] >>= eval
      where cond = cs!!0
            condClause (List [Atom "else", b]) = return $ List [Bool True, b]
            condClause (List [p,b]) = do x <- eval cond
                                         y <- eval p
                                         q <- eqv [x, y]
                                         case q of
                                           Bool _ -> return $ List [q, b]
                                           _ -> throwError $ TypeMismatch "bool" q
            condClause v = throwError $ TypeMismatch "(pred body)" v
            f = \(List [p,b]) -> case p of
                                   (Bool False) -> True
                                   _ -> False
~~~
{: .language-haskell}

至此以后的章节，都没有课后习题了。

## Building a REPL: Basic I/O

书中使用了 Parsec 模块的 try ，所以这里不导入 IO 的 try。

~~~
import IO hiding (try)
IO Monad 本身已经输出状态了，所以这里用 Monad Then 传递：

flushStr :: String -> IO ()
flushStr str = putStr str >> hFlush stdout
~~~
{: .language-haskell}

这个 IO Monad then 的运用非常漂亮

~~~
readPrompt :: String -> IO String
readPrompt prompt = flushStr prompt >> getLine
~~~
{: .language-haskell}

摘录： That's why we write their types using the type variable "m", and include the type constraint "Monad m =>"

~~~
until_ :: Monad m => (a -> Bool) -> m a -> (a -> m ()) -> m ()
until_ pred prompt action = do 
  result <- prompt
  if pred result 
     then return ()
     else action result >> until_ pred prompt action
~~~
{: .language-haskell}

## Adding Variables and Assignment: Mutable State in Haskell

State Monad 对于简单的有状态应用非常好用。

这里使用 state threads，具体而言是 Data.IORef。

~~~
import Data.IORef

type Env = IORef [(String, IORef LispVal)]
建立空环境的 nullEnv 函数：

nullEnv :: IO Env
nullEnv = newIORef []
~~~
{: .language-haskell}

将异常 lift 为自定义的 IOThrowsError

~~~
liftThrows :: ThrowsError a -> IOThrowsError a
liftThrows (Left err) = throwError err
liftThrows (Right val) = return val
~~~
{: .language-haskell}

原文摘录：Methods in typeclasses resolve based on the type of the expression, so throwError and return (members of MonadError and Monad, respectively) take on their IOThrowsError definitions.

函数 runIOThrows 对 trapError 进行了 IO 包装：

~~~
runIOThrows :: IOThrowsError String -> IO String
runIOThrows action = runErrorT (trapError action) >>= return . extractValue
~~~
{: .language-haskell}

函数 lookup 对键值对序列做 key 查询，返回 Maybe value。所以这里利用 const True 做了一个 isBound 查询函数。

~~~
isBound :: Env -> String -> IO Bool
isBound envRef var = readIORef envRef >>= return . maybe False (const True) . lookup var
~~~
{: .language-haskell}

对 IORef 环境的经典读写操作：

~~~
getVar :: Env -> String -> IOThrowsError LispVal
getVar envRef var = do env <- liftIO $ readIORef envRef
                       maybe (throwError $ UnboundVar "Getting an unbound variable" var)
                             (liftIO . readIORef) (lookup var env)

setVar :: Env -> String -> LispVal -> IOThrowsError LispVal
setVar envRef var value = do env <- liftIO $ readIO envRef
                             maybe (throwError $ UnboundVar "Setting an unboud variable" var)
                                   (liftIO . (flip writeIORef value))
                                   (lookup var env)
                             return value
~~~
{: .language-haskell}

在 bindVars 函数中，利用了 Monadic 管道操作技巧：

~~~
bindVars :: Env -> [(String, LispVal)] -> IO Env
bindVars envRef bindings = readIORef envRef >>= extendEnv bindings >>= newIORef
    where extendEnv bindings env = liftM (++ env) (mapM addBinding bindings)
          addBinding (var, value) = do ref <- newIORef value
                                       return (var, ref)
~~~
{: .language-haskell}

eval 中传入 env 参数以保持状态，其中 cond 和 case 基本上算重写了一遍……

~~~
eval :: Env -> LispVal -> IOThrowsError LispVal
eval env val@(String _) = return val
eval env val@(Number _) = return val
eval env val@(Bool _) = return val
eval env (Atom id) = getVar env id
eval env (List [Atom "quote", val]) = return val
eval env (List [Atom "if", pred, conseq, alt]) =
    do result <- eval env pred
       case result of
         Bool True -> eval env conseq
         Bool False -> eval env alt
         otherwise -> throwError $ TypeMismatch "bool" pred
eval env (List ((Atom "cond"):cond:cs)) = do
  result <- condClause env cond
  if (f result) 
     then eval' env result
     else eval env (List ((Atom "cond"):cs))
    where condClause env (List [Atom "else", b]) = return $ List [Bool True, b]
          condClause env (List [p,b]) = do q <- eval env p
                                           case q of
                                             Bool _ -> return $ List [q,b]
                                             _ -> throwError $ TypeMismatch "bool" q
          condClause env v = throwError $ TypeMismatch "(pred body)" v
          f = \(List [p,b]) -> case p of
                                 (Bool True) ->  True
                                 (Bool False) -> False
          eval' = \env (List [p, b]) -> eval env b                                      
eval env (List ((Atom "case"):c:cond:cs)) = do
  x <- eval env c
  result <- condClause env cond x
  if (f result)
     then eval' env result
     else eval env (List ((Atom "case"):x:cs))
    where condClause env (List [Atom "else", b]) x = return $ List [Bool True, b]
          condClause env (List [p,b]) x = do y <- eval env p
                                             q <- return $ eqv [x, y]
                                             (\e -> let v = extractValue e
                                                    in case v of
                                                         Bool _ -> return $List [v, b]
                                                         _ -> throwError $ TypeMismatch "bool" v) q
          condClause env v x = throwError $ TypeMismatch "(pred body)" v
          f = \(List [p,b]) -> case p of
                                 (Bool False) -> False
                                 _ -> True
          eval' = \env (List [p, b]) -> eval env b
eval env (List [Atom "set!", Atom var, form]) =
    eval env form >>= setVar env var
eval env (List [Atom "define", Atom var, form]) =
    eval env form >>= defineVar env var
eval env (List (Atom func : args)) = mapM (eval env) args >>= liftThrows.apply func
eval env badForm = throwError $ BadSpecialForm "Unrecognized special form" badForm
~~~
{: .language-haskell}

因为要保持环境，单行语句和 REPL 的执行函数分成了两个：

~~~
runOne :: String -> IO ()
runOne expr = nullEnv >>= flip evalAndPrint expr

runRepl :: IO ()
runRepl = nullEnv >>= until_ (== "quit") (readPrompt "Lisp>>> ") . evalAndPrint

main :: IO ()
main = do args <- getArgs
          case length args of
              0 -> runRepl
              1 -> runOne $ args !! 0
              otherwise -> putStrLn "Program takes only 0 or 1 argument"
~~~
{: .language-haskell}

##  Defining Scheme Functions: Closures and Environments

代码示例基本没什么错误
函数的定义和函数执行是两个相关的内容。

扩展后的 LispVal：

~~~
data LispVal = Atom String
             | List [LispVal]
             | DottedList [LispVal] LispVal
             | Number Integer
             | Float Double
             | String String
             | Bool Bool
             | Character Char
             | Ratio Rational
             | Complex (Complex Double)
             | Vector (Array Int LispVal)
             | PrimitiveFunc ([LispVal] -> ThrowsError LispVal)
             | Func {params :: [String], vararg :: (Maybe String),
                     body :: [LispVal], closure :: Env}
             | Nil ()
~~~
{: .language-haskell}

函数定义涉及参数名、参数列、函数体和闭包。新的 apply 实现了函数定义和执行：

函数解析代码中先判定了形参与实参是否一致（如果没有指定动态参数， 给出的实参个数又与形参不同，则报错）。然后利用 curry 方法生成 该函数的函数实现。在各操作（绑定作用域、参数和解析函数体）的过程 中，以 Monad bind 串起各部分。注意这里 bindVars 接受的第一 个函数是 closure ，在 eval 中我们可以看到 closure 是 外部的 env。在传值过程中，env 复制为函数 closure。

~~~
apply :: LispVal -> [LispVal] -> IOThrowsError LispVal
apply (PrimitiveFunc func) args = liftThrows $ func args
apply (Func params varargs body closure) args =
    if num params /= num args && varargs == Nothing
       then throwError $ NumArgs (num params) args
       else (liftIO $ bindVars closure $ zip params args) >>= bindVarArgs varargs >>= evalBody
    where remainingArgs = drop (length params) args
          num = toInteger . length
          evalBody env  = liftM last $ mapM (eval env) body
          bindVarArgs arg env = case arg of
                                  Just argName -> liftIO $ bindVars env [(argName, List $ remainingArgs)]
                                  Nothing -> return env
~~~
{: .language-haskell}

函数相关的 eval 实现。

~~~
eval env (List (Atom "define" : List (Atom var : params) : body)) =
    makeNormalFunc env params body >>= defineVar env var
eval env (List (Atom "define" : DottedList (Atom var : params) varargs : body)) =
    makeVarargs varargs env params body >>= defineVar env var
eval env (List (Atom "lambda" : List params : body)) =
    makeNormalFunc env params body
eval env (List (Atom "lambda" : DottedList params varargs : body)) =
    makeVarargs varargs env params body
eval env (List (Atom "lambda" : varargs@(Atom _) : body)) =
    makeVarargs varargs env [] body
eval env (List (function : args)) = do func <- eval env function
                                       argVals <- mapM (eval env) args
                                       apply func argVals
~~~
{: .language-haskell}

在 eval 中使用到的函数构造代码：

~~~
makeFunc varargs env params body = return $ Func (map showVal params) varargs body env
makeNormalFunc = makeFunc Nothing
makeVarargs = makeFunc . Just . showVal
~~~
{: .language-haskell}

环境初始化代码进行了进一步的扩展，将所有的 primitive 封装为函数。这部分实现的简洁有力：

~~~
primitiveBindings :: IO Env
primitiveBindings = nullEnv >>= (flip bindVars $ map makePrimitiveFunc primitives)
    where makePrimitiveFunc (var, func) = (var, PrimitiveFunc func)

runOne :: String -> IO ()
runOne expr = primitiveBindings >>= flip evalAndPrint expr

runRepl :: IO ()
runRepl = primitiveBindings >>= until_ (== "quit") (readPrompt "Lisp>>> ") . evalAndPrint
~~~
{: .language-haskell}

最后这几章的内容中，数次出现了 flip 操作。

## Creating IO Primitives: File I/O

这一章代码没什么争议
因为输入的参数不同（数组和数值的区别），io 的指令集是独立实现的：

~~~
ioPrimitives :: [(String, [LispVal] -> IOThrowsError LispVal)]
ioPrimitives = [("apply", applyProc),
                ("open-input-file", makePort ReadMode),
                ("open-output-file", makePort WriteMode),
                ("close-input-port", closePort),
                ("close-output-port", closePort),
                ("read", readProc),
                ("write", writeProc),
                ("read-contents", readContents),
                ("read-all", readAll)]
~~~
{: .language-haskell}

其底层的文件操作都是对 haskell 库的浅封装：

~~~
applyProc :: [LispVal] -> IOThrowsError LispVal
applyProc [func, List args] = apply func args
applyProc (func : args) = apply func args

readProc :: [LispVal] -> IOThrowsError LispVal
readProc [] = readProc [Port stdin]
readProc [Port port] = (liftIO $ hGetLine port) >>= liftThrows . readExpr

writeProc :: [LispVal] -> IOThrowsError LispVal
writeProc [obj] = writeProc [obj, Port stdout]
writeProc [obj, Port port] = liftIO $ hPrint port obj >> (return $ Bool True)

readContents :: [LispVal] -> IOThrowsError LispVal
readContents [String filename] = liftM String $ liftIO $ readFile filename

makePort :: IOMode -> [LispVal] -> IOThrowsError LispVal
makePort mode [String filename] = liftM Port $ liftIO $ openFile filename mode

closePort :: [LispVal] -> IOThrowsError LispVal
closePort [Port port] = liftIO $ hClose port >> (return $ Bool True)
closePort _ = return $ Bool False

load :: String -> IOThrowsError [LispVal]
load filename = (liftIO $ readFile filename) >>= liftThrows . readExprList

readAll :: [LispVal] -> IOThrowsError LispVal
readAll [String filename] = liftM List $ load filename
~~~
{: .language-haskell}

为了处理单语句和多语句，代码解析也进行了抽象，分别实现具体的操作

~~~
readOrThrow :: Parser a -> String -> ThrowsError a
readOrThrow parser input = case parse parser "lisp" input of
                             Left err -> throwError $ Parser err
                             Right val -> return val

readExpr = readOrThrow parseExpr
readExprList = readOrThrow (endBy parseExpr spaces)
~~~
{: .language-haskell}

环境初始化，引入指令集的实现，针对 IO 作了扩展：

~~~
primitiveBindings :: IO Env
primitiveBindings = nullEnv >>= (flip bindVars $ map (makeFunc IOFunc) ioPrimitives
                                        ++ map (makeFunc PrimitiveFunc) primitives)
    where makeFunc constructor (var, func) = (var, constructor func)
~~~
{: .language-haskell}

交互过程做了相应的调整

~~~
runOne :: [String] -> IO ()
runOne args = do
  env <- primitiveBindings >>= flip bindVars [("args", List $ map String $ drop 1 args)]
  (runIOThrows $ liftM show $ eval env (List [Atom "load", String (args !! 0)]))
              >>= hPutStrLn stderr

runRepl :: IO ()
runRepl = primitiveBindings >>= until_ (== "quit") (readPrompt "Lisp>>> ") . evalAndPrint

main :: IO ()
main = do args <- getArgs
          if null args then runRepl else runOne $ args
~~~
{: .language-haskell}

## Towards a Standard Library: Fold and Unfold

这一章只有 scheme 的代码和内置库实现，留待想要学习 sheme 语言的时候深入。

## Conclusion & Further Resources

附录跳过

## 总结

[原书在线阅读](http://jonathan.tang.name/files/scheme_in_48/tutorial/overview.html)

[参考答案](http://en.wikibooks.org/wiki/Write_Yourself_a_Scheme_in_48_Hours/Answers)

时间和精力有限，我没有认真的去做每一道题，而仅仅是把在线答案中的每一个 调试验证了一遍。示例中还是有一些小错误的，如果只是复制，无法顺利编译运 行，有一些我在笔记中有提到，附件中是我自己的练习代码。

从我个人体验来讲，用 ghci 来练习，要比编译后调试要快捷得多。不过从第 六章 Evalution 2 开始，需要 -XExistentialQuantification 选项。 如果你和我一样使用 emacs 的 ghci 集成 shell ，没办法用 ghci -XExistentialQuantification 形式启动交互环境，可以在 ghci 中执行 :set -XExistentialQuantification 加载这一选项。

这本书不应该作为 Haskell 学习的第一本书，甚至也不一定应该是第二本书。 学习此书最好有初步的 lisp/scheme 知识。

我的阅读过程还是比较粗糙的，建议在遇到不能完全理解的地方，用 ghci 调试 一下。

作为第一本书，[Yet Another Haskell Tutorial](http://www.cs.utah.edu/~hal/docs/daume02yaht.pdf) 更系统，内容循序渐进。

作为第二本书，[Real World Haskell](http://www.realworldhaskell.org/) 更有实践性，曲线比较平滑。

本书应该作为从初阶到中阶的一本阶段总结辅导。或者，除非你有 scheme/lisp 基础，对第一本书也学的比较顺利……

