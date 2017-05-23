# 5.2. OS 對 NUMA 的支援

為了支援 NUMA 機器，OS 必須將記憶體分散式的性質納入考量。舉例來說，若是一個行程執行在一個給定的處理器上，被指派給行程位址空間的實體 RAM 理想上應該要來自本地記憶體。否則每條指令都必須為了程式碼與資料去存取遠端的記憶體。有些僅存於 NUMA 機器的特殊情況要被考慮進去。DSO 的文字區段（text segment）在一台機器的實體 RAM 中通常正好出現一次。但若是 DSO 被所有 CPU 上的行程與執行緒用到（例如，像 `libc` 這類基本的執行期函式庫），這表示並非一些、而是全部的處理器都必須擁有遠端的位址。OS 理想上會將這種 DSO「映像（mirror）」到每個處理器的實體 RAM 中，並使用本地的副本。這並非一種最佳化，而是個必要條件，而且通常難以實作。它可能沒有被支援、或者僅以有限的形式支援。

為了避免情況變得更糟，OS 不該將一個行程或執行緒從一個節點遷移到另一個。OS 應該已經試著避免在一般的多處理器機器上遷移行程，因為從一個處理器遷移到另一個處理器意味著快取內容遺失了。若是負載分配需要將一個行程或執行緒遷出一個處理器，OS 通常能夠挑選任一個擁有足夠剩餘容量的新處理器。在 NUMA 環境中，新處理器的選擇受到了稍微多一些的限制。對於行程正在使用的記憶體，新選擇的處理器不該有比舊的處理器還高的存取成本；這限制了目標的清單。若是沒有符合可用標準的空閒處理器，OS 除了遷移到記憶體存取更為昂貴的處理器以外別無他選。

在這種情況下，有兩種可能的方法。首先，可以期盼這種情況是暫時的，而且行程能夠被遷回到一個更合適的處理器上。或者，OS 也能夠將行程的記憶體遷移到更靠近新使用的處理器的實體分頁上。這是個相當昂貴的操作。可能得要複製大量的記憶體，儘管不必在一個步驟中完成。當發生這種情況的時候，行程必須––至少短暫地––中止，以正確地遷移對舊分頁的修改。有了讓分頁遷移有效又快速，有整整一串其它的必要條件。簡而言之，OS 應該避免它，除非它是真的有必要的。

一般來說，不能夠假設在一台 NUMA 機器上的所有行程都使用等量的記憶體，使得––由於遍及各個處理器的行程的分佈––記憶體的使用也會被均等地分配。事實上，除非執行在機器上的應用程式非常特殊（在 HPC 世界中很常見，但除此之外則否），不然記憶體的使用是非常不均等的。某些應用程式會使用巨量的記憶體，其餘的幾乎不用。若總是分配產生請求的處理器本地的記憶體，這將會––或早或晚––造成問題。系統最終將會耗盡執行大行程的節點本地的記憶體。

為了應對這些嚴重的問題，記憶體––預設情況下––不會只分配給本地的節點。為了利用所有系統的記憶體，預設策略是條帶化（stripe）記憶體。這保證了所有系統記憶體的同等使用。作為一個副作用，有可能變得能在處理器之間自由遷移行程，因為––平均而言––對於所有用到的記憶體的存取成本沒有改變。由於很小的 NUMA 因子，條帶化是可以接受的，但仍不是最好的（見 5.4 節的數據）。

這是個幫助系統避免嚴重問題、並在普通操作下更為能夠預測的劣化（pessimization）。但它降低了整體的系統效能，在某些情況下尤為顯著。這即是 Linux 允許每個行程選擇記憶體分配規則的原因。一個行程能夠為它自己以及它的子行程選擇不同的策略。我們將會在第六節介紹能用於此的介面。
