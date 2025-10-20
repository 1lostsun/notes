`growslice` - выделяет новое **хранилище** (backing store) для **среза (slice)**.

### Функция `growslice` принимает следующие аргументы: 

- `oldPtr` - указатель на текущий массив, лежащий в основе среза;
- `newLen` - новая длина среза (`=oldlen + num`);
- `oldCap` - старая вместимость среза (capacity);
- `num` - количество добавляемых элементов;
- `et` - тип элемента среза (element type).

### Возвращаемые значения: 

- `newPtr` - указатель на новое выделенное хранилище;
- `newLen` - новая длина среза;
- `newCap` - вместимость нового хранилища.

---
### Условия и поведение: 

- `{go} uint(newLen) > uint(oldCap)`;
- Предполагается, что исходная длина среза равна `newLen - num`.

Функция **выделяет новое хранилище**, рассчитанное **как минимум на `newLen`** элементов.
Существующие элементы в диапазоне `[o, oldLen)` **копируются** в новое хранилище.

Элементы в диапазоне `[oldLen, newLen)` **не инициализируются** самой функцией `growslice`
(однако, если тип элементов содержит указатели, они будут **обнулены**).

Их инициализацией **должен** заниматься **вызвавший код**.
Элементы в диапазоне `[newLen, newCap)` всегда обнуляются.

---
### Особенности соглашения о вызове (calling convention):

- Оно сделано намеренно необычным, чтобы **упростить код**,
  генерируемый компилятором;
- В частности, функция **принимает** и **возвращает** `newLen`, чтобы:
	- Старое значение длины (`oldLen`) **не оставалось "живым"**
	 (не требовало сохранения и восстановления),
	- Новое значение длины **возвращалось напрямую**, 
	 также без необходимости хранения.

Функция `growslice` **должна быть внутренней деталью реализации**,  
однако на практике **многие популярные пакеты используют её напрямую**  
с помощью `linkname`.

---
```go
// nextslicecap computes the next appropriate slice length.
func nextslicecap(newLen, oldCap int) int {  
  newcap := oldCap  
  doublecap := newcap + newcap  
  if newLen > doublecap {  
   return newLen  
  }  
  
  const threshold = 256  
  if oldCap < threshold {  
   return doublecap  
  }  
  for {  
   // Transition from growing 2x for small slices  
   // to growing 1.25x for large slices. This formula   
   // gives a smooth-ish transition between the two.   
   newcap += (newcap + 3*threshold) >> 2  
  
   // We need to check `newcap >= newLen` and whether `newcap` overflowed.  
   // newLen is guaranteed to be larger than zero, hence   
   // when newcap overflows then `uint(newcap) > uint(newLen)`.  
   // This allows to check for both with the same comparison.   
   if uint(newcap) >= uint(newLen) {  
    break  
   }  
  }  
  
  // Set newcap to the requested cap when  
  // the newcap calculation overflowed.  
  if newcap <= 0 {  
   return newLen  
  }  
  
  return newcap  
}
```