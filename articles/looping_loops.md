---
title: "Распутывая циклы Ansible"
layout: none
render_with_liquid: false  # Более явное отключение
---

# Распутывая циклы Ansible

Написать данный опус навеяла статья на хабре [Распутывая Ansible Loops](https://habr.com/ru/post/526526/). 
Вообще с циклами у ansible на мой взгляд не задалось.  Никаких тебе конструкций вида for и while. Официальная документация довольно-таки развернутая, но немного в ней не хватает элементарных примеров. Их и постараюсь привести.

## Перебираем списки

Cписок наверное самое простое и частоиспользуемое что можно перебрать в ansible. Перебирать элементы списка можно с использованием loop и Jinja фильтров:

```yaml
- name: print list, reversed list and sorted list
  debug:
    msg: "List is {{ list_row }} \nReverse list is {{ list_row|reverse }} \nSorted list is {{ list_row|sort }}"

- name: print item of sorted and reversesd list
  debug:
    msg: '{{ item }}'
  loop: '{{ list_row|sort|reverse }}'

TASK [print list, reversed list and sorted list] 
MSG:
List is ['row51', 'row2', 'row13', 'row4']
Reverse list is ['row4', 'row13', 'row2', 'row51']
Sorted list is ['row13', 'row2', 'row4', 'row51']

TASK [print item of sorted and reversesd list] 
MSG:
row51
MSG:
row4
MSG:
row2
MSG:
row13
```

## Перебираем таблицы

Теперь разберемся с простыми циклами по таблице. Ограничения python понятны - это не mathlab, заточенный под работу с матрицами, и каждая таблица-матрица-тензор удобнее всего представляется списком из списков из списков...
Пробежимся по строкам, столбцам и элементам - слева направо сверху вниз и изменив порядок.

```yaml
vars:
  table:
    - ['a', 'b', 'c']
    - ['d', 'e', 'f']
    - ['g', 'h', 'i']
```

```yaml
- name: print rows
  debug:
    msg: '{{ item }}'
  loop: '{{ table|list }}'

- name: print column
  debug:
    msg: '{{ item }}'
  loop: '{{ table[0]|zip(*table[1:])|list }}'

- name: print elements left2rigth up2down
  debug:
    msg: '{{ item }}'
  loop: '{{ table|list|flatten(levels=1) }}'

- name: print elements up2down left2rigth
  debug:
    msg: '{{ item }}'
  loop: '{{ table[0]|zip(*table[1:])|list|flatten(levels=1) }}'
```

Результат

```yaml
TASK [print rows] *
MSG:
['a', 'b', 'c']
MSG:
['d', 'e', 'f']
MSG:
['g', 'h', 'i']

TASK [print column] *
MSG:
['a', 'd', 'g']
MSG:
['b', 'e', 'h']
MSG:
['c', 'f', 'i']

TASK [print elements left2rigth up2down]
MSG:
a
MSG:
b
MSG:
c
MSG:
d
MSG:
e
MSG:
f
MSG:
g
MSG:
h
MSG:
i

TASK [print elements up2down left2rigth]
MSG:
a
MSG:
d
MSG:
g
MSG:
b
MSG:
e
MSG:
h
MSG:
c
MSG:
f
MSG:
i
```

## Перебираем строки

Почему строки после списков? Да потому что строка в ansible неитерируема из коробки и надо сделать приседание чтобы получить хорошо знакомый нам список.

```yaml
- debug: var=test_string

- name: Iterate to symvol
  debug:
    msg: '{{ item }}'
  loop: '{{ test_string|list }}'

- name: Iterate to word
  debug:
    msg: '{{ item }}'
  loop: '{{ test_string.split(' ') }}'

TASK [debug]
ok: [kafka-1.my.domain] => {
    "test_string": "hello World!"
}

TASK [Iterate to symvol]
MSG:
h
MSG:
e
MSG:
l
MSG:
l
MSG:
o
MSG:

MSG:
W
MSG:
o
MSG:
r
MSG:
l
MSG:
d
SG:
!

TASK [Iterate to word]
MSG:
hello
MSG:
World!
```

## Перебираем словари

Словари как и списки можно перебрать, сославшись на каждое значение словаря, сортировав словарь, или извлекая из словаря значение key и соответствующее value.
Стоит обратить внимание что фильтр dictsort преобразует словарь в список и сортрует его по значению ключей.

```yaml
vars:
  dict_pass:
    broker : ['pass_11', 'pass_12']
    root : ['pass_21', 'pass_22']
    client : ['pass_31', 'pass_32']
```

```yaml
- name: print dict items
  debug:
    msg: '{{ item }}'
  loop: '{{ dict_pass|dict2items }}'

- name: print sorted dict items
  debug:
    msg: '{{ item }}'
  loop: '{{ dict_pass|dictsort }}'

- name: print dict items
  debug:
    msg: '{{ item.key }} and {{ item.value.0 }} and {{ item.value.1 }}'
  loop: '{{ dict_pass|dict2items }}'
```

результат

```yaml
TASK [print dict items] *
MSG:
{'key': 'broker', 'value': ['pass_11', 'pass_12']}
MSG:
{'key': 'root', 'value': ['pass_21', 'pass_22']}
MSG:
{'key': 'client', 'value': ['pass_31', 'pass_32']}

TASK [print sorted dict items]
MSG:
['broker', ['pass_11', 'pass_12']]
MSG:
['client', ['pass_31', 'pass_32']]
MSG:
['root', ['pass_21', 'pass_22']]

TASK [print dict items] *
MSG:
broker and pass_11 and pass_12
MSG:
root and pass_21 and pass_22
MSG:
client and pass_31 and pass_32
```

## Перебираем группы

Наш инвентарь будет выглядеть так (не судите строго).

```ini
[all]
[all:children]
test
cluster
kafka
[kafka]
kafka-1.my.domain
kafka-2.my.domain
[test]
nginx-1.my.domain
nginx-2.my.domain
kafka-5.my.domain
kafka-3.my.domain
[cluster]
[cluster:children]
postgres
etcd
[postgres]
postgres-1.my.domain
postgres-2.my.domain
postgres-3.my.domain
[etcd_nodes]
etcd-1.my.domain
etcd-2.my.domain
etcd-3.my.domain

[all:vars]
ansible_ssh_port='7722'
ansible_user='ansible-allsudo'
ansible_password='Ansible_pa$$word'
```

Попробуем вывести значение groups.

```yaml
- name: print groups
  debug: var=groups

TASK [print groups]
    "all": [
           "kafka-1.my.domain",
            "kafka-2.my.domain",
            "nginx-1.my.domain",
            "nginx-2.my.domain",
            "kafka-5.my.domain",
            "kafka-3.my.domain",
            "postgres-1.my.domain",
            "postgres-2.my.domain",
            "postgres-3.my.domain"
        ],
        "cluster": [
            "postgres-1.my.domain",
            "postgres-2.my.domain",
            "postgres-3.my.domain"
        ],
        "etcd_nodes": [
            "etcd-1.my.domain",
            "etcd-2.my.domain",
            "etcd-3.my.domain"
        ],
        "kafka": [
            "kafka-1.my.domain",
            "kafka-2.my.domain"
        ],
        "postgres": [
            "postgres-1.my.domain",
            "postgres-2.my.domain",
            "postgres-3.my.domain"
        ],
        "test": [
            "nginx-1.my.domain",
            "nginx-2.my.domain",
            "kafka-5.my.domain",
            "kafka-3.my.domain"
        ],
        "ungrouped": []
    }
}
```

Итак, имеем:

groups - словарь, элементами которого являются списки. Проверяем:

```yaml
- name: print groups as dict
  debug:
    msg: '{{ item }}'
  loop: '{{ groups|dict2items }}'

TASK [print groups]
MSG:
{'key': 'all', 'value': ['kafka-1.my.domain', 'kafka-2.my.domain', 'nginx-1.my.domain', 'nginx-2.my.domain', 'kafka-5.my.domain', 'kafka-3.my.domain', 'postgres-1.my.domain', 'postgres-2.my.domain', 'postgres-3.my.domain']}
MSG:
{'key': 'ungrouped', 'value': []}
MSG:
{'key': 'kafka', 'value': ['kafka-1.my.domain', 'kafka-2.my.domain']}
MSG:
{'key': 'test', 'value': ['nginx-1.my.domain', 'nginx-2.my.domain', 'kafka-5.my.domain', 'kafka-3.my.domain']}
MSG:
{'key': 'cluster', 'value': ['postgres-1.my.domain', 'postgres-2.my.domain', 'postgres-3.my.domain']}
MSG:
{'key': 'postgres', 'value': ['postgres-1.my.domain', 'postgres-2.my.domain', 'postgres-3.my.domain']}
MSG:
{'key': 'etcd_nodes', 'value': ['etcd-1.my.domain', 'etcd-2.my.domain', 'etcd-3.my.domain']}
```

Значит и обращаться с groups надо соответствующим образом - ьакже как и со словарем.

Еще один способ итерироваться по хостам - использовать магическую переменную ansible_play_hosts.

```yaml
- name: Create virtual group
  add_host:
    name: '{{ item|regex_replace('[.].*','')}}"
    groups:
      - firstnodes
    ansible_host: '{{ item }}'
  when: item | regex_search('.*1')
  loop: '{{ ansible_play_hosts }}'
  register: result

- debug:
    var: groups.firstnodes
  when: result.results[0].changed
  run_once: true

TASK [Create virtual group]
omain)
main)
kipping: [kafka-1.my.domain] => (item=kafka-5.my.domain)
skipping: [kafka-1.my.domain] => (item=kafka-3.my.domain)
omain)
kipping: [kafka-1.my.domain] => (item=postgres-3.my.domain)

TASK [debug]
ok: [kafka-1.my.domain] => {
    "groups.firstnodes": [
        "kafka-1",
        "nginx-1",
        "postgres-1"
    ]
}
```

Стоит обратить внимание на то что первая таска с модулем *add_host* обрабатывается лишь однажды, несмотря на то что нет *run_once: true*. А *groups.firstnodes* является списком.

Ну и посмотрим что внутри вновь созданной группы firstnodes:

```yaml
- hosts: firstnodes
  gather_facts: false
  tasks:
  - debug:
      msg: '{{ inventory_hostname }} and {{ ansible_host }}'

PLAY [firstnodes]
TASK [debug]
ok: [kafka-1] => {}
MSG:
kafka-1 and kafka-1.my.domain
ok: [nginx-1] => {}
MSG:
nginx-1 and nginx-1.my.domain
ok: [postgres-1] => {}
MSG:
postgres-1 and postgres-1.my.domain
```
