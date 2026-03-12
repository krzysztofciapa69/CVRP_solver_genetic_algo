# Problem Definition: CVRP with a Global Permutation

The problem addressed is a specific variant of the Capacitated Vehicle Routing Problem (CVRP) where a global permutation of clients (often referred to as a "giant tour") is provided as input.

It is notable that `.lcvrp` files contain ideal (or the best-found-so-far) permutations. Consequently, splitting the permutation into contiguous segments provides a mathematically optimal partitioning into routes or groups. We can achieve this using the Bellman-Ford algorithm ($O(n^2)$), or the faster Split algorithm ($O(n)$) authored by Thibaut Vidal.



However, in version 3 of the competition package, the permutation from the file is shuffled, which yields a geometrically unordered and rather low-quality visit sequence. Nevertheless, Split remains a highly powerful tool, though its application must be precisely adapted to the specific characteristics of the problem.

---

## Key Operators Used

### 1) Crossover

* **Geometric (Neighbor-based):** Selects a random client from Parent 1 and assigns the route/group IDs of its nearest neighbors to the offspring. The remaining assignments are then inherited from Parent 2.
* **SREX (Sequence Routing Crossover):** Exchanges entire routes. Approximately half of the routes are inherited from Parent 1, and then as many non-conflicting routes as possible are taken from Parent 2. The remaining clients that caused conflicts are inserted into routes using a regret-3 heuristic.

### 2) Mutations

* **MicroSplit:** Executes the Split algorithm described above but within a smaller window. This allows the algorithm to hit a "good" segment of the permutation and partition it optimally.
* **MergeSplit:** Merges two routes/groups and performs an optimal Split on their clients. This is highly effective for "shuffling" and re-splitting, as ranks (sequence within the permutation) can easily become deadlocked.
* **Ruin & Recreate (RR):** A standard operator in CVRP variants. It removes the route assignment of a client and its nearest neighbors (the exact amount depends on the disruption intensity), and then reinserts them while taking into account route capacities and the number of geometric neighbors in other routes.
* **Adaptive LNS (ALNS):** The core idea is similar to standard RR, but the destruction phase relies on a dynamically selected strategy. Available destruction strategies target clusters within the permutation, over-capacitated routes, or the worst routes in terms of distance and fill rate.
* **ReturnMinimizer:** Eliminates truck returns; for example, it reduces a 130% fill rate down to 100%.
* **EliminateReturns:** Increases the fill rate (packing density). It removes the tail ends of routes where the vehicle deadheads (returns to the depot empty, e.g., utilization < 70%). From the evaluator's perspective, one route filled at 200% yields the exact same fitness result as two vehicles packed at 100%, provided the permutation segments do not overlap.

### 3) Local Search

*This component selects the applied operator based on its historical performance via Adaptive Operator Selection (AOS).*

* **VND (Variable Neighborhood Descent):** For standard instance sizes, it relocates a client from one route to another or swaps two clients between routes. For feasible moves—those that do not violate capacity constraints—it uses delta evaluation. For infeasible moves, it simulates the cost of the entire route.
* **Path Relinking (PR):** Calculates the topological difference between an individual and the guide solution (the best individual on a given island) and attempts to connect these solutions to find a superior intermediate configuration.
* **Ejection Chains:** A sequence of chained client displacements and swaps.
* **3-Swap, 4-Swap:** Selects geometrically close clients and tests their various combinations (evaluating $3!$ and $4!$ moves is computationally acceptable).

---

## Architecture: Heterogeneous Island Model

The architecture utilizes a heterogeneous island model where each island is handled by a separate thread and is assigned a distinct operational role. Islands with even IDs act as "Explorers," while odd-ID islands are "Exploiters."

* **Explorer:** Searches for new, promising areas of the solution space. It features a high mutation probability, fast VND (Relocate + Swap), and a larger population size.
* **Exploiter (Exploit):** Polishes and uncovers deep local optima. It utilizes a lower mutation probability and heavy VND operations (Ejection Chains, Path Relinking, 3-Swap, 4-Swap).

### Diversity Management

Preventing premature convergence is critical and is managed using the Broken Pairs Distance (BPD) metric. In the event of population stagnation, a catastrophic mutation is triggered. After 5 unsuccessful catastrophes, the island transfers its individuals to other islands, completely restarts from scratch, and blocks incoming migration for 60 seconds.

### Migration

Migration operates asynchronously and depends on the health of a specific island. Aside from fitness criteria, an adequate structural difference in the BPD metric is a strict prerequisite for an individual entering an island. Broadcast mechanisms are also implemented: when any island finds a solution significantly better than the current global best, it broadcasts it to the remaining network of islands.

---

## Advanced Concepts

### RoutePool & Beam Search

A RoutePool classifies and stores the best routes found so far, an idea heavily inspired by Mixed Integer Programming (MIP) and Column Generation. Routes are evaluated based on their load fill rate and fitness contribution. Periodically, a Beam Search algorithm assembles a "Frankenstein" solution from these cached, non-overlapping routes. 

If the resulting solution meets quality thresholds, it is introduced into the population; additionally, there is a 10% probability it is force-injected regardless of quality. This continuously supplies the population with high-quality building blocks, accelerating convergence during Local Search and recombination processes (like SREX). In practice, injecting a "Frankenstein" solution after a long period of stagnation frequently leads to finding a new global best immediately. Gene migration between islands is also active, exploiting the fact that each island searches a distinct area of the solution space.

### Deduplication & Hashing

64-bit caching mechanisms are enforced in three critical bottlenecks: during offspring generation, during the evaluation of complete solutions, and for caching individual routes within the Evaluator. Utilizing diverse hashing methods mitigates the risk of hash collisions. Route-level hashing is particularly vital for VND performance, where the entire route must be strictly simulated for unsafe moves.

### Initialization Methods

To guarantee high initial diversity, varying initialization methods are deployed, such as Random, Chunked, or K-Center. The K-Center implementation deliberately avoids using centroids (unlike K-Means), making it strictly viable for explicit cost matrix instances. Another mechanism involves creating a temporary permutation using a Double Bridge move, which is subsequently subjected to the Split algorithm, ensuring the initial routes maintain rigorous mathematical coherence.











Polska wersja:
Można zauważyć, pliki .lcvrp zawierają idealne ( albo najlepsze dotychczas znalezione ) permutacje. Dzięki temu, podzielenie permutacji na ciągłe fragmenty da nam optymalnie matematyczny podział do grupy. Możemy wykorzystać do tego algorytm Bellmana-Forda (O(n^2)), lub szybszy algorytm Split (O(n)) autorstwa Thibauta Vidala. Jednakże, w 3 wersji paczki konkursowej permutacja z pliku jest tasowana, przez co dostajemy nieuporządkowaną geometrycznie i raczej słabej jakości kolejność odwiedzin. Niemniej jednak uważam, że Split jest nadal potężnym narzędziem, jednak używanie go trzeba dostosować do specyfiki problemu.

Najważniejsze stosowane operatory:

1) Crossover:

\- geometryczny, bazujący na sąsiadach. Losuje losowego klienta od rodzica 1 a następnie przepisuje do dziecka id grup jego k najbliższych sasiadow. Reszta jest wtedy brana od rodzica 2.

\- SREX. Wymienia całe trasy. Około połowa tras jest brana od rodzica 1, następnie od rodzica 2 brane jest jak najwięcej tras, które nie kolidują z istniejącymi. Pozostali klienci, którzy kolidowali są dobierani do grup za pomocą heurystyki regret 3.

2) Mutacje:

\- MicroSplit. Realizuje wyżej opisany algorytm, ale na mniejszym oknie. Dzięki czemu możemy trafić na "dobry" kawałek permutacji i go optymalnie podzielić.

\- MergeSplit. Łączy 2 grupy i na ich klientach wykonuje optymalny podzial. Jest dobry również do "tasowania" i dzielenia ponownie, ponieważ rangi (kolejność w permutacji) mogą się zakleszczać.

\- RuinRecreate - standard w wariantach CVRP. Usuwa przydział do grup klienta i jego k najbliższych sąsiadów ( k zależy od intensywności) a następnie przydziela ich z powrotem z uwzględnieniem pojemności grup oraz ilości sąsiadów geometrycznych w innych grupach.

\- AdaptiveLNS - idea jest podobna do standardowego RR, ale moment niszczenia jest zależny od wybranej strategii. Możemy niszczyć m.in. klastry w permutacji, przepełnione grupy oraz najgorsze trasy pod względem dystansu i zapełnienia

\- ReturnMinimizer - usuwa powroty z ciężarówek np. wypełnienie 130% zmienia na 100%.

\- EliminateReturns - Zwiększenie upakowania (fill-rate). Usuwa końcówki tras, gdzie samochód wraca do bazy wioząc powietrze (np. wykorzystanie < 70%). Warto zaznaczyć, że z perspektywy evaluatora jedna trasa wypełniona w 200% da taki sam rezultat co 2 wozy upakowane w 100% ( pod warunkiem że kawałki permutacji się nie nałożą)

3) LocalSearch - prawdopodobnie najważniejszy element, wybiera stosowany operator w zależności od jego skuteczności (AOS) :

\- VND. Dla standardowych rozmiarów instancji przenosi klienta z 1 grupy do drugiej, lub zamienia 2 klientów grupami. Dla bezpiecznych ruchów, czyli takich które nie przekroczą pojemności, sotsuje delta evaluation, a dla niebezpiecznych symuluje koszt całej trasy.

\- Path Relinking sprawdza różnice między osobnikiem i guide solution (najlepszym na danej wyspie) i stara się połączyć te rozwiązania i znaleźć w konsekwencji lepsze.

\- Ejection Chains - jest to łańcuhcowa zamiana klientów

\- 3Swap, 4Swap. Wybiera geometrycznie bliskich klientów i testuje ich różne kombinacje (3! oraz 4! ruchów są akceptowalne wydajnościowo)

W projekcie postawiłem na heterogeniczny model wyspowy. Każda wyspa jest obslugiwana przez osobny wątek i ma osobną rolę / zadania. Wyspy o id parzystych to "Explorerzy" a nieparzystych nazywam "Exploatorami"

Explorator. Jak nazwa wskazuje, jego zadaniem jest szukanie nowych, obiecujących kawałków przestrzeni rozwiązań - duża szansa na mutacje, szybkie vnd z relocate + swap, większa populacja.

Exploit. Poleruje i odkrywa głębokie optima lokalne - mniejsza szansa na mutacje, ciężkie vnd z ejection chains, PR 3,4 Swap.

Zarządzanie różnorodnością - jest to bardzo ważny element aby uniknąć przedwczesnej zbieżności. Oparty jest na metryce złamanych pair ( Broken Pairs Distance ). W przypadku "wymarcia populacji" stosowana jest katastrofa. Po 5 nieudanych katastrofach wyspa przerzuca swoich osobników na inne wyspy, zaczyna kompletnie od nowa i blokuje migrację na 60 sekund.

Migracja - jest asynchroniczna w zależności od zdrowia konkretnej wyspy oprócz fitnessu, warunkiem wejścia na wyspę jest odpowiednia różnica w metryce BPD. Wprowadzone są również broadcasty - gdy któraś wyspa znajdzie znacząco lepsze rozwiązanie od poprzedniego globalnego current best, wysyła je do pozostałych wysp.

Inne, ciekawe pomysły:

RoutePool - klasyfikuje i zapisuje najlepsze dotychczasowe trasy. Idea została zaczerpnięta z MIP. Trasy oceniane są na podstawie załadowania tras jak najlepszego fitnessu. Następnie, co jakiś czas składa z nich "Frankensteina" (poprzez połączenie nienakładających się ze sobą tras) z nich za pomocą Beamsearcha. Wpuszczany jest do populacji jeśli jest wystarczająco dobry, ale również z 10% szansą siłowo wstrzykuje go do populacji. Dzięki temu, dostarcza dobre cechy do rozwiązań, co w procesach localsearch i mieszania np. crossoverem SREX może dać swietne wyniki. Analizując działanie programu, wielokrotnie widoczne było że po długim czasie stagnacji "Frankenstein" był wstrzykiwany do populacji, a od razu potem na tej wyspie znaleziono globalny best. Wprowadzona jest również migracja genów między wyspami, ponieważ w teorii każda wyspa przeszukuje inną przestrzeń rozwiązań.

Deduplikacja:

Mechanizmy cachowania (na 64 bitach) wprowadzone są w 3 miejscach - przy tworzeniu dzieci, przy evaluowaniu całych rozwiązań oraz do cachowania pojedynczych tras w moim Evaluatorze. Wykorzystywanie różnych sposobów hashowania zapewnia większe bezpieczeństwo kolizji hashy, a hashowanie tras jest kluczowe dla działania VND, gdzie symulowane jest cała trasa w przypadku niebezpiecznych ruchów.

Sposoby inicjalizajci:

Dla zachowania różnorodności, wybierane metody inicjalizacji osobników są różne - np. Random, Chunked czy K-Center (nie wykorzystuje centroidow jak K-means dzięki czemu działa dla instancji explicit). Inny ciekawy mechanizm stosowany to tworzenie tymczasowej permutacji za pomocą double bridge a następnie poddawanie jej algorytmowi split. Dzięki temu, przynajmniej w teorii, trasy mają jakiś matematyczny sens.
