Синхронные скрипты — это плохо, поскольку они заставляют браузер блокировать 
построение DOM дерева: сначала нужно получить скрипт, выполнить его, а только 
после этого продолжать обработку оставшейся части страницы. Это, конечно, не 
должно стать для вас новостью, и это также является причиной, по которой мы, 
как евангелисты, продвигали использование асинхронных сценариев.
