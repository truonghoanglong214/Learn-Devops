# Phase 3 — Git & Version Control

## Mục tiêu
- Quản lý mã nguồn chuyên nghiệp, làm việc nhóm được.
- Áp dụng quy trình branching đúng chuẩn doanh nghiệp.
- Chuẩn bị nền tảng cho CI/CD (Phase 7).

---

## Kiến thức sẽ học

- **Git cơ bản** (init, add, commit, log, diff, status).
- **Branching & merging, conflict resolution** — làm việc song song không đè code nhau.
- **Remote (GitHub/GitLab), push/pull/fetch, Pull/Merge Request**.
- **Branching strategy** (Git Flow, GitHub Flow, Trunk-based).
- **`.gitignore`, semantic commit, tag & release**.
- **Bảo vệ branch, code review** — văn hóa kỹ thuật doanh nghiệp.

---

## Kiến thức chi tiết

### 1. Git Internals — Hiểu để dùng đúng

Hầu hết developer dùng Git như một công cụ magic: gõ lệnh, code được lưu, xong. Nhưng để không bị bất ngờ khi conflict, mất code, hay gặp lỗi lạ, bạn cần hiểu Git hoạt động bên dưới như thế nào.

#### Ba vùng làm việc của Git

Git quản lý code của bạn qua ba vùng riêng biệt. Hiểu ba vùng này là chìa khóa để không bao giờ bị mất code.

**Working Directory (Thư mục làm việc)**

Đây là nơi bạn đang thực sự viết code. Tất cả các file bạn có thể nhìn thấy trong thư mục dự án đều nằm ở đây. Khi bạn sửa một file, thay đổi đó tồn tại trong working directory và chưa được Git theo dõi cho đến khi bạn làm thêm bước tiếp theo.

Trạng thái của file trong working directory:
- **Untracked**: file mới, Git chưa bao giờ biết đến file này
- **Modified**: file đã được Git theo dõi nhưng có thay đổi chưa được stage
- **Deleted**: file đã bị xóa nhưng chưa stage việc xóa đó

**Staging Area / Index (Vùng chuẩn bị)**

Đây là bước trung gian giữa working directory và repository. Khi bạn chạy `git add`, bạn đang "stage" các thay đổi — tức là nói với Git: "những thay đổi này sẽ nằm trong commit tiếp theo của tôi".

Staging area cực kỳ hữu ích vì:
- Bạn có thể làm nhiều thay đổi nhưng chỉ commit một phần
- Bạn có thể review kỹ những gì sắp commit trước khi thực sự commit
- Bạn có thể stage từng phần của một file (dùng `git add -p`)

Staging area được lưu trong file `.git/index` — đây là lý do nó còn được gọi là "index".

**Local Repository (Kho lưu trữ cục bộ)**

Đây là database Git lưu toàn bộ lịch sử commit của bạn, nằm trong thư mục `.git/`. Khi bạn chạy `git commit`, các thay đổi từ staging area được đóng gói thành một commit object và lưu vĩnh viễn vào repository.

"Vĩnh viễn" ở đây có nghĩa là: một khi đã commit, rất khó mất code. Git còn giữ các commit "mồ côi" (orphaned commits) trong một thời gian nhất định trước khi garbage collect.

**Remote Repository (Kho lưu trữ từ xa)**

Là bản copy của repository nằm trên server — GitHub, GitLab, Bitbucket, hoặc tự host. Remote phục vụ hai mục đích chính:
- **Backup**: nếu máy local hỏng, code vẫn an toàn
- **Collaboration**: nhiều người có thể cùng làm việc trên cùng codebase

Remote không phải là "ground truth" về lịch sử commit — mỗi bản copy đều bình đẳng về mặt kỹ thuật. Nhưng theo convention, remote `origin` (thường là GitHub) được coi là nguồn chính thức.

#### Luồng dữ liệu qua ba vùng

```
Working Directory  -->  Staging Area  -->  Local Repository  -->  Remote
     (edit)          (git add)           (git commit)          (git push)
                                                    <--
                                               (git fetch / pull)
```

#### Git Object Model — Bên trong .git/

Git lưu trữ mọi thứ dưới dạng "objects". Có bốn loại object:

**Blob (Binary Large Object)**

Blob lưu nội dung thuần túy của một file. Quan trọng: blob không lưu tên file hay permissions, chỉ lưu nội dung. Hai file khác nhau có cùng nội dung sẽ dùng chung một blob — đây là cách Git tiết kiệm dung lượng.

```
Ví dụ: file src/app.js có nội dung "console.log('hello')"
Git tạo blob object với SHA-1: a8f5f167f44f4964e6c998dee827110c0f15a4b0
```

**Tree**

Tree đại diện cho một thư mục. Nó chứa danh sách các entries, mỗi entry có:
- Mode (permissions: 100644 cho file, 040000 cho directory)
- Object type (blob hoặc tree)
- SHA-1 hash của object
- Tên file/thư mục

Tree cho phép Git tái hiện lại toàn bộ cấu trúc thư mục của dự án tại bất kỳ thời điểm nào.

**Commit**

Commit object chứa:
- Pointer tới root tree (snapshot của toàn bộ project)
- Pointer tới parent commit(s) — tạo thành lịch sử tuyến tính
- Author name, email, timestamp
- Committer name, email, timestamp (có thể khác author khi cherry-pick)
- Commit message

Đây là lý do tại sao Git có thể "travel in time" — mỗi commit đều biết cha nó là ai.

**Tag**

Tag object là pointer được đặt tên trỏ tới một commit cụ thể, thường dùng để đánh dấu release. Annotated tag (dùng `git tag -a`) tạo một object riêng với tagger info và message. Lightweight tag chỉ là một pointer đơn giản.

#### SHA-1 Hash

Mỗi object Git đều có một SHA-1 hash 40 ký tự hexadecimal duy nhất, được tính từ nội dung của object đó. Tính chất quan trọng:

- **Deterministic**: cùng nội dung luôn cho cùng hash
- **Unforgeable**: không thể tạo collision trong thực tế
- **Content-addressed**: hash chính là "địa chỉ" của content trong Git's object store
- **Integrity check**: nếu file bị corrupt, hash sẽ không khớp

Bạn không cần gõ toàn bộ 40 ký tự — Git chấp nhận prefix đủ dài để unique (thường 7 ký tự):
```bash
git show a8f5f16   # thay vì git show a8f5f167f44f4964e6c998dee827110c0f15a4b0
```

#### Cấu trúc thư mục .git/

```
.git/
├── objects/           # Tất cả blob, tree, commit, tag objects
│   ├── pack/          # Packfiles (objects được nén sau khi có nhiều objects)
│   ├── info/
│   └── ab/            # Subdirectory = 2 ký tự đầu của SHA-1
│       └── cdef...    # Phần còn lại của SHA-1
├── refs/              # Pointers (branches, tags, remotes)
│   ├── heads/         # Local branches
│   │   ├── main
│   │   └── feature/auth
│   ├── remotes/       # Remote tracking branches
│   │   └── origin/
│   └── tags/          # Tags
├── HEAD               # Pointer tới current branch hoặc commit
├── config             # Repository-level config
├── COMMIT_EDITMSG     # Message của commit cuối cùng
├── index              # Staging area (binary format)
├── FETCH_HEAD         # Kết quả của git fetch gần nhất
├── ORIG_HEAD          # Backup của HEAD trước destructive operations
└── MERGE_HEAD         # Khi đang trong quá trình merge
```

**HEAD — Pointer đặc biệt**

`HEAD` luôn trỏ tới commit hiện tại của bạn. Thông thường HEAD là "symbolic ref" — tức là trỏ tới một branch, và branch đó trỏ tới commit:

```
HEAD --> refs/heads/main --> commit abc1234
```

Khi bạn ở trạng thái "detached HEAD" (sau `git checkout abc1234`), HEAD trỏ thẳng tới commit thay vì branch. Nếu bạn tạo commit mới trong trạng thái này và không tạo branch, commit đó sẽ bị mồ côi và có thể bị xóa bởi garbage collector.

---

### 2. Setup Ban Đầu

Trước khi dùng Git, cần cấu hình một lần cho toàn bộ máy tính.

#### Cấu hình bắt buộc

```bash
# Tên hiển thị trong mỗi commit — dùng tên thật của bạn
git config --global user.name "Nguyen Van A"

# Email phải khớp với email GitHub/GitLab để GitHub link được commit với account
git config --global user.email "nguyenvana@example.com"
```

#### Cấu hình khuyến nghị

```bash
# Từ Git 2.28+, tên nhánh mặc định khi git init là 'main' thay vì 'master'
git config --global init.defaultBranch main

# Dùng VS Code làm default editor (thay vì vim)
# --wait: Git đợi cho đến khi bạn đóng file trong VS Code
git config --global core.editor "code --wait"

# Khi git pull: dùng merge thay vì rebase (an toàn hơn cho người mới)
git config --global pull.rebase false

# Luôn hiển thị màu sắc trong output
git config --global color.ui auto

# Tắt cảnh báo CRLF trên Windows (nếu đang dùng Windows)
git config --global core.autocrlf true

# Cấu hình diff tool (optional)
git config --global diff.tool vscode
git config --global difftool.vscode.cmd "code --wait --diff $LOCAL $REMOTE"
```

#### Kiểm tra cấu hình

```bash
# Xem tất cả config và nguồn gốc của từng setting
git config --list --show-origin

# Xem một setting cụ thể
git config user.email

# Xem config file trực tiếp
cat ~/.gitconfig
```

#### Phân cấp cấu hình Git

Git có ba cấp config, cấp sau override cấp trước:
1. **System** (`/etc/gitconfig`): áp dụng cho tất cả user trên máy
2. **Global** (`~/.gitconfig`): áp dụng cho tất cả repo của user hiện tại
3. **Local** (`.git/config`): chỉ áp dụng cho repo cụ thể này

```bash
# Override email cho một repo cụ thể (ví dụ: work project dùng work email)
git config --local user.email "work@company.com"
```

#### SSH Key Setup (khuyến nghị hơn HTTPS)

Dùng SSH key để không cần nhập password mỗi lần push/pull:

```bash
# Tạo SSH key (chọn Ed25519, mạnh hơn RSA)
ssh-keygen -t ed25519 -C "your.email@example.com"

# Thêm vào SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Copy public key để paste vào GitHub Settings > SSH Keys
cat ~/.ssh/id_ed25519.pub

# Test kết nối
ssh -T git@github.com
```

---

### 3. Git Cơ Bản — Mỗi Lệnh Có Ví Dụ Thực Tế

#### Khởi tạo Repository

```bash
# Tạo repo mới trong thư mục hiện tại
git init

# Tạo repo mới trong thư mục mới tên 'shoplite'
git init shoplite

# Clone repo từ GitHub về máy
git clone https://github.com/user/shoplite.git

# Clone vào thư mục với tên khác
git clone https://github.com/user/shoplite.git my-shoplite

# Shallow clone: chỉ lấy commit gần nhất, nhanh hơn nhiều
# Dùng khi bạn chỉ cần code hiện tại, không cần full history
git clone --depth 1 https://github.com/user/shoplite.git

# Clone một branch cụ thể
git clone --branch develop --single-branch https://github.com/user/shoplite.git
```

#### Kiểm tra Trạng Thái

```bash
# Xem trạng thái tổng quan
git status

# Xem trạng thái ngắn gọn (compact)
git status -s
# Output: M = modified, A = added, ? = untracked, D = deleted

# Xem changes trong working directory (chưa stage)
git diff

# Xem changes đã stage (sẽ vào commit tiếp theo)
git diff --staged
# Alias: git diff --cached (tương đương)

# So sánh giữa hai branches
git diff main..feature/auth

# So sánh với commit cụ thể
git diff abc1234..HEAD

# Chỉ xem tên file thay đổi, không cần chi tiết
git diff --name-only main..feature/auth

# Thống kê số dòng thêm/xóa
git diff --stat main..feature/auth
```

#### Staging Changes

```bash
# Stage một file cụ thể
git add src/components/Login.js

# Stage toàn bộ một thư mục
git add src/

# Stage tất cả thay đổi (modified + new + deleted)
# CẨNTHAN: có thể vô tình stage file không mong muốn
git add -A
git add --all  # tương đương

# Stage chỉ files đã tracked (không stage untracked files mới)
git add -u

# Interactive staging: review từng "hunk" của diff và quyết định có stage không
# Đây là kỹ năng quan trọng để tạo commits nguyên tử (atomic commits)
git add -p
git add --patch  # tương đương
# Trong interactive mode: y = stage, n = skip, s = split hunk, e = edit manually, q = quit

# Stage một phần của một file cụ thể
git add -p src/app.js
```

**Tại sao dùng `git add -p`?**

Giả sử bạn đang làm feature "user authentication" nhưng tiện tay sửa luôn một bug nhỏ không liên quan. Thay vì commit cả hai vào một commit, bạn có thể:
1. `git add -p` để stage chỉ phần liên quan đến auth feature
2. `git commit -m "feat(auth): add login form validation"`
3. `git add -p` để stage phần bug fix
4. `git commit -m "fix(ui): correct button alignment on mobile"`

Kết quả là lịch sử commit sạch, dễ đọc, dễ revert từng phần.

#### Commit

```bash
# Commit với message trên command line
git commit -m "feat(auth): add user login with JWT"

# Mở editor để viết message dài hơn (multi-line)
git commit
# Dòng đầu: subject (tối đa 72 ký tự)
# Dòng 2: để trống
# Dòng 3+: body (giải thích WHY, không phải WHAT)

# Stage tất cả tracked files và commit trong một lệnh
# (không stage untracked files mới)
git commit -am "fix(cart): prevent negative quantity"

# Sửa commit cuối cùng (CHỈ dùng khi CHƯA push lên remote)
# Có thể thay đổi message và/hoặc thêm files vào commit
git commit --amend -m "feat(auth): add user login with JWT token refresh"

# Sửa commit cuối nhưng giữ nguyên message (chỉ thêm files còn quên stage)
git add forgotten-file.js
git commit --amend --no-edit
```

**Quy tắc vàng: ĐỪNG amend commits đã push lên shared branch.** Amend thay đổi SHA-1 của commit, gây ra diverged history cho những người đã pull commit đó.

#### Xem Lịch Sử

```bash
# Xem log đẹp nhất — one line, graph, tất cả branches, có tên branch
git log --oneline --graph --all --decorate

# Xem chi tiết diff của từng commit
git log -p

# Giới hạn số commit hiển thị
git log -10

# Tìm commits của một tác giả
git log --author="Nguyen Van A"

# Tìm commits trong khoảng thời gian
git log --since="2 weeks ago" --until="1 day ago"

# Tìm commits có chứa keyword trong message
git log --grep="authentication"

# Tìm commits có thay đổi liên quan đến một string trong code
# (pickaxe search — rất hữu ích khi debug)
git log -S "getUserById"

# Xem commits trên feature branch nhưng không có trên main
git log main..feature/auth

# Xem chi tiết một commit cụ thể
git show abc1234

# Xem nội dung file tại một commit cụ thể
git show abc1234:src/app.js

# Xem ai đã viết từng dòng của file và commit nào
git blame src/controllers/auth.js

# Xem blame với context (3 dòng trước/sau mỗi match)
git blame -L 50,80 src/controllers/auth.js
```

#### Stash — Lưu Work Tạm Thời

Stash hữu ích khi bạn đang làm dở feature nhưng cần switch sang branch khác để fix urgent bug.

```bash
# Lưu tất cả changes vào stash (kể cả staged)
git stash

# Lưu với tên mô tả (khuyến nghị để dễ nhớ)
git stash push -m "wip: implement product search pagination"

# Stash cả untracked files (files mới chưa add)
git stash push -u -m "wip: product search including new files"

# Xem danh sách stash entries
git stash list
# Output: stash@{0}: On feature/search: wip: implement product search pagination
#         stash@{1}: On main: wip: hotfix attempt

# Apply stash mới nhất và xóa khỏi stash list
git stash pop

# Apply một stash cụ thể và xóa
git stash pop stash@{1}

# Apply stash nhưng KHÔNG xóa (giữ lại trong list để dùng ở nơi khác)
git stash apply stash@{0}

# Xem diff của stash trước khi apply
git stash show -p stash@{0}

# Xóa một stash entry
git stash drop stash@{0}

# Xóa toàn bộ stash list
git stash clear

# Tạo branch mới từ stash (hữu ích khi stash có conflict với current branch)
git stash branch feature/search-pagination stash@{0}
```

---

### 4. Branching & Merging

#### Quản Lý Branches

```bash
# Xem tất cả local branches (* = current branch)
git branch

# Xem tất cả branches bao gồm remote-tracking branches
git branch -a

# Xem branches với thông tin last commit
git branch -v

# Xem branches đã merge vào current branch (an toàn để xóa)
git branch --merged

# Xem branches chưa merge (CẦN THẬN trước khi xóa)
git branch --no-merged

# Tạo branch mới (chưa switch)
git branch feature/user-auth

# Switch sang branch (modern syntax — Git 2.23+)
git switch feature/user-auth

# Tạo branch mới và switch ngay (most common workflow)
git switch -c feature/product-search

# Switch về branch trước đó (tương đương cd -)
git switch -

# Tạo branch từ một commit cụ thể thay vì HEAD
git switch -c hotfix/login-crash abc1234

# Tạo branch tracking remote branch
git switch -c feature/auth origin/feature/auth
# Hoặc ngắn hơn:
git switch --track origin/feature/auth

# Đổi tên branch hiện tại
git branch -m new-name

# Xóa branch đã merged (safe)
git branch -d feature/completed-feature

# Force delete branch chưa merged (mất code nếu chưa backup)
git branch -D feature/abandoned-feature

# Xóa remote-tracking branch reference
git remote prune origin
```

#### Merge

```bash
# Merge feature branch vào current branch (thường là main hoặc develop)
git switch main
git merge feature/user-auth

# Các loại merge:

# 1. Fast-forward merge (mặc định khi có thể):
# Nếu main không có commit mới kể từ khi tạo feature branch,
# Git chỉ di chuyển pointer main tới tip của feature branch.
# Không tạo merge commit. History tuyến tính.

# 2. No-fast-forward merge (khuyến nghị cho feature branches):
# Luôn tạo merge commit, preserve branch topology trong history.
# Dễ revert một feature hoàn chỉnh bằng cách revert merge commit.
git merge --no-ff feature/user-auth -m "merge: add user authentication feature"

# 3. Squash merge:
# Gộp tất cả commits của feature branch thành một commit duy nhất trong main.
# Giữ main history clean nhưng mất granular history của feature branch.
git merge --squash feature/user-auth
git commit -m "feat(auth): add complete user authentication system"
```

#### Giải Quyết Merge Conflict từng bước

Conflict xảy ra khi hai branches sửa cùng một phần của cùng một file.

```bash
# Bước 1: Cố gắng merge
git switch main
git merge feature/payment-integration
# OUTPUT: CONFLICT (content): Merge conflict in src/controllers/order.js
# Automatic merge failed; fix conflicts and then commit the result.

# Bước 2: Xem những file nào có conflict
git status
# both modified: src/controllers/order.js

# Bước 3: Mở file conflicted
# Trong file bạn sẽ thấy conflict markers:
```

```
<<<<<<< HEAD
// Code từ branch main (current branch)
const calculateTotal = (items) => {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
};
=======
// Code từ branch feature/payment-integration
const calculateTotal = (items, discount = 0) => {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  return subtotal * (1 - discount);
};
>>>>>>> feature/payment-integration
```

```bash
# Bước 4: Quyết định code nào đúng
# Option A: Giữ HEAD (main)
# Option B: Giữ feature/payment-integration
# Option C: Kết hợp cả hai (thường là trường hợp này)

# Kết quả sau khi resolve — xóa tất cả conflict markers:
```

```javascript
// Code kết hợp — giữ discount parameter từ feature branch
const calculateTotal = (items, discount = 0) => {
  const subtotal = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  return subtotal * (1 - discount);
};
```

```bash
# Bước 5: Stage file đã resolve
git add src/controllers/order.js

# Bước 6: Nếu còn file conflict khác, tiếp tục resolve và add
# Bước 7: Hoàn thành merge
git commit
# Git tự động tạo merge commit message — có thể giữ nguyên hoặc edit

# NẾU muốn hủy merge và quay về trạng thái trước
git merge --abort
```

**Công cụ hỗ trợ resolve conflict:**
- VS Code: built-in merge editor với GUI đẹp
- IntelliJ/WebStorm: excellent 3-way merge view
- `git mergetool`: mở configured diff tool

#### Rebase

Rebase là cách khác để tích hợp changes từ branch này vào branch khác. Thay vì tạo merge commit, rebase "viết lại" lịch sử bằng cách replay từng commit của bạn lên đầu target branch.

```bash
# Scenario: bạn đang trên feature/search, muốn cập nhật với thay đổi mới nhất từ main

# Cách 1: Merge (creates merge commit, preserves timeline)
git switch feature/search
git merge main

# Cách 2: Rebase (no merge commit, linear history, NHƯNG rewrites history)
git switch feature/search
git rebase main
# Git sẽ: tạm thời remove commits của feature/search,
# fast-forward tới tip của main, rồi replay từng commit của bạn

# NẾU có conflict trong rebase:
# 1. Resolve conflict trong file
# 2. git add resolved-file
# 3. git rebase --continue  (tiếp tục replay commit tiếp theo)
# 4. Hoặc: git rebase --abort  (hủy toàn bộ rebase)

# QUY TẮC VÀNG: KHÔNG rebase commits đã push lên shared branch
# Rebase rewrites SHA-1 → diverged history → chaos cho teammates
```

**Interactive Rebase — Dọn Dẹp Lịch Sử Trước Khi Merge**

```bash
# Mở interactive rebase cho 3 commits gần nhất
git rebase -i HEAD~3

# Mở editor với nội dung như sau:
# pick abc1234 feat: add search input
# pick def5678 fix typo
# pick ghi9012 wip: still working on this
#
# Các actions có thể dùng:
# pick (p)    : giữ nguyên commit
# reword (r)  : giữ commit nhưng thay đổi message
# edit (e)    : dừng lại để amend commit này
# squash (s)  : gộp vào commit trước, kết hợp messages
# fixup (f)   : gộp vào commit trước, bỏ message của commit này
# drop (d)    : xóa commit hoàn toàn
# exec (x)    : chạy shell command sau commit này

# Ví dụ: squash 3 commits thành 1
# Thay đổi editor thành:
# pick abc1234 feat: add search input
# squash def5678 fix typo
# squash ghi9012 wip: still working on this

# Sau khi save, Git mở editor lần nữa để viết combined commit message
```

#### Cherry-pick

```bash
# Lấy một commit cụ thể từ branch khác và apply vào branch hiện tại
# Hữu ích khi: hotfix cần apply vào cả main và release branch

git switch release/1.0
git cherry-pick abc1234   # SHA-1 của hotfix commit từ main

# Cherry-pick một range của commits
git cherry-pick abc1234..def5678

# Cherry-pick nhưng không tạo commit ngay (stage thay đổi để bạn edit trước)
git cherry-pick --no-commit abc1234
```

#### Undo và Reset

Đây là phần nhiều người sợ nhất nhưng thực ra rất logic nếu hiểu ba vùng làm việc.

```bash
# Unstage một file (ngược lại của git add)
git restore --staged src/app.js
# Lệnh cũ (vẫn hoạt động): git reset HEAD src/app.js

# Discard changes trong working directory (KHÔNG THỂ UNDO)
git restore src/app.js
# Lệnh cũ: git checkout -- src/app.js

# Xem tất cả references và commits gần đây (safety net)
git reflog
# Hữu ích để recover sau git reset --hard

# git reset — có ba chế độ:
# --soft: di chuyển HEAD, giữ changes ở staging area
git reset --soft HEAD~1
# Dùng khi: muốn undo commit nhưng giữ lại code đã stage để commit lại

# --mixed (DEFAULT): di chuyển HEAD, unstage changes, giữ trong working dir
git reset --mixed HEAD~1
git reset HEAD~1   # tương đương
# Dùng khi: muốn undo commit và tổ chức lại thành nhiều commits nhỏ hơn

# --hard: di chuyển HEAD, XÓA TẤT CẢ CHANGES (working dir + staging)
git reset --hard HEAD~1
# Dùng khi: muốn xóa hoàn toàn commit và thay đổi
# NGUY HIỂM: không thể undo trừ khi dùng git reflog

# git revert — AN TOÀN cho shared branches
# Tạo commit MỚI undo lại thay đổi của commit được chỉ định
# Không thay đổi history → an toàn khi đã push
git revert HEAD        # revert commit cuối
git revert abc1234     # revert một commit cụ thể
git revert --no-commit HEAD~3..HEAD  # revert 3 commits nhưng stage thay đổi, không commit ngay
```

**Khi nào dùng reset vs revert?**
- `reset`: chưa push, chỉ trên local branch của bạn
- `revert`: đã push lên shared branch, cần undo một cách "công khai"

---

### 5. Remote Operations

#### Quản Lý Remote

```bash
# Xem danh sách remotes
git remote -v
# Output: origin  https://github.com/user/shoplite.git (fetch)
#         origin  https://github.com/user/shoplite.git (push)

# Thêm remote (khi init local repo và muốn link với GitHub)
git remote add origin https://github.com/user/shoplite.git

# Đổi URL của remote (ví dụ: từ HTTPS sang SSH)
git remote set-url origin git@github.com:user/shoplite.git

# Thêm remote thứ hai (ví dụ: upstream khi fork)
git remote add upstream https://github.com/original-owner/shoplite.git

# Xóa remote
git remote remove upstream

# Đổi tên remote
git remote rename origin github
```

#### Fetch, Pull, Push

```bash
# FETCH: download changes từ remote nhưng KHÔNG tự động merge
# An toàn nhất: bạn có thể review trước khi merge
git fetch origin

# Fetch tất cả remotes
git fetch --all

# Fetch và xóa remote-tracking branches không còn tồn tại trên remote
git fetch --all --prune

# Sau khi fetch, xem những gì đã thay đổi
git log HEAD..origin/main --oneline
git diff HEAD..origin/main

# PULL: fetch + merge (hoặc fetch + rebase nếu config pull.rebase = true)
git pull

# Pull và rebase thay vì merge (cleaner history)
git pull --rebase

# Chỉ pull branch cụ thể
git pull origin develop

# PUSH: upload commits lên remote
# First push: -u sets tracking relationship
git push -u origin feature/user-auth

# Subsequent pushes (sau khi đã set tracking)
git push

# Push tất cả local branches
git push --all origin

# Push tags
git push --tags
git push origin v1.0.0  # push một tag cụ thể

# Xóa remote branch
git push origin --delete feature/old-feature
# Hoặc: git push origin :feature/old-feature

# Force push (NGUY HIỂM — chỉ dùng trên branch của riêng bạn)
# --force-with-lease: an toàn hơn --force
# Sẽ fail nếu remote branch đã có commits mới mà bạn chưa fetch
# Tránh ghi đè code của người khác
git push --force-with-lease origin feature/my-branch

# KHÔNG BAO GIỜ: git push --force origin main
```

#### Tracking Branches

```bash
# Xem tracking relationship của mỗi branch
git branch -vv

# Set upstream tracking cho branch hiện tại
git branch --set-upstream-to=origin/main main

# Sau khi set tracking, git status sẽ hiện thị:
# "Your branch is ahead of 'origin/main' by 2 commits."
```

---

### 6. Branching Strategy

#### GitHub Flow — Chiến Lược cho ShopLite

GitHub Flow là branching strategy đơn giản nhất, phù hợp nhất với ShopLite trong giai đoạn đầu.

**Nguyên tắc cơ bản:**
- `main` là branch duy nhất tồn tại lâu dài
- `main` luôn luôn trong trạng thái có thể deploy
- Mỗi feature, fix, improvement đều bắt đầu bằng việc tạo branch từ `main`
- Sau khi hoàn thành, tạo Pull Request vào `main`
- Sau khi PR được merge, deploy ngay lập tức

**Workflow chi tiết:**

```bash
# Bước 1: Cập nhật main
git switch main
git pull origin main

# Bước 2: Tạo feature branch với tên mô tả
git switch -c feature/product-search-with-filters

# Bước 3: Làm việc và commit thường xuyên
# (commit nhỏ, atomic, meaningful messages)
git add src/components/SearchBar.js
git commit -m "feat(search): add search bar component"
git add src/api/products.js
git commit -m "feat(search): add product search API endpoint"
git add tests/search.test.js
git commit -m "test(search): add unit tests for search functionality"

# Bước 4: Push lên remote (ngay từ đầu để backup và visibility)
git push -u origin feature/product-search-with-filters

# Bước 5: Mở Pull Request trên GitHub
# Title: "feat: add product search with filters"
# Body: mô tả what, why, how to test

# Bước 6: Request review từ teammate
# Reviewer có thể comment trực tiếp trên từng dòng code

# Bước 7: Address feedback
git add src/components/SearchBar.js
git commit -m "fix(search): handle empty search query edge case"
git push

# Bước 8: CI checks pass, approval received → Merge PR
# (trên GitHub UI, chọn "Squash and merge" hoặc "Create a merge commit")

# Bước 9: Xóa feature branch (GitHub có thể làm tự động)
git switch main
git pull origin main
git branch -d feature/product-search-with-filters
```

**Tại sao GitHub Flow phù hợp với ShopLite:**
- Solo developer hoặc team nhỏ (2-4 người)
- Deploy thường xuyên (nhiều lần trong ngày)
- Không có release schedule phức tạp
- Đơn giản, dễ onboard member mới

#### Git Flow — Khi Nào Cần Dùng

Git Flow phù hợp khi team lớn hơn, có release cycle rõ ràng (ví dụ: release mỗi 2 tuần), hoặc cần maintain nhiều version cùng lúc.

**Cấu trúc branches:**

```
main          — Production code, chỉ merge từ release/* và hotfix/*
develop       — Integration branch, base cho tất cả features
feature/*     — Tính năng mới, tách từ develop, merge về develop
release/*     — Chuẩn bị release, tách từ develop khi feature-complete
hotfix/*      — Fix bug khẩn cấp trên production, tách từ main
```

**Workflow ví dụ:**

```bash
# Bắt đầu feature mới
git switch -c feature/payment-integration develop

# ... làm việc ...

# Merge feature vào develop
git switch develop
git merge --no-ff feature/payment-integration

# Khi develop sẵn sàng để release
git switch -c release/1.2.0 develop
# ... chỉ bug fixes, không thêm features mới ...
# Bump version number, update CHANGELOG

# Release hoàn chỉnh
git switch main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

git switch develop
git merge --no-ff release/1.2.0

git branch -d release/1.2.0

# Hotfix trên production
git switch -c hotfix/fix-payment-crash main
# ... fix ...
git switch main
git merge --no-ff hotfix/fix-payment-crash
git tag -a v1.2.1 -m "Hotfix: fix payment crash"

git switch develop
git merge --no-ff hotfix/fix-payment-crash
git branch -d hotfix/fix-payment-crash
```

**Trunk-Based Development (TBD)**

Tất cả developer commit thẳng vào `main` (hoặc trunk) thường xuyên, thường nhiều lần mỗi ngày. Feature branches tồn tại không quá 1-2 ngày. Dùng "feature flags" để ẩn tính năng chưa hoàn thiện.

Phù hợp với team có CI/CD mạnh, culture testing tốt, và developer senior.

---

### 7. .gitignore Đầy Đủ cho ShopLite

`.gitignore` quyết định những file nào Git sẽ không theo dõi. Quan trọng vì: bảo mật (không commit secrets), performance (không commit node_modules), và cleanliness (không commit OS/IDE artifacts).

```gitignore
# =============================================================
# ShopLite .gitignore
# =============================================================

# Node.js
# =============================================================
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.npm
.yarn-integrity
.pnp.*
.yarn/cache
.yarn/unplugged
.yarn/build-state.yml

# Build outputs
dist/
build/
out/
.next/
.nuxt/
.cache/

# Environment variables — KHÔNG BAO GIỜ COMMIT
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
# Nhưng COMMIT .env.example (template không có values thật)

# =============================================================
# Python (nếu có service Python)
# =============================================================
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
ENV/
env/
*.egg-info/
.eggs/
dist/
*.egg
.pytest_cache/
.coverage
htmlcov/
.tox/
.mypy_cache/

# =============================================================
# OS-generated files
# =============================================================
# macOS
.DS_Store
.AppleDouble
.LSOverride
._*

# Windows
Thumbs.db
Thumbs.db:encdata
ehthumbs.db
Desktop.ini
$RECYCLE.BIN/

# Linux
*~

# =============================================================
# IDE và Editor
# =============================================================
# JetBrains (IntelliJ, WebStorm, PyCharm, etc.)
.idea/
*.iws
*.iml
*.ipr
out/

# VS Code — CHÚ Ý: chỉ ignore personal settings
.vscode/*.code-workspace
# NHƯNG NÊN COMMIT các file sau để đồng bộ team:
# .vscode/settings.json (project settings như tab size, formatters)
# .vscode/extensions.json (recommended extensions)
# .vscode/launch.json (debug configurations)

# Vim
*.swp
*.swo
*~

# Emacs
*~
\#*\#

# =============================================================
# Docker
# =============================================================
.docker/
docker-compose.override.yml

# =============================================================
# Logs
# =============================================================
*.log
logs/
npm-debug.log*
combined.log
error.log

# =============================================================
# Testing
# =============================================================
coverage/
.nyc_output/
jest-results/

# =============================================================
# Database
# =============================================================
*.db
*.sqlite
*.sqlite3

# =============================================================
# SECRETS — QUAN TRỌNG NHẤT
# Không bao giờ commit bất kỳ file nào chứa secrets
# =============================================================
*.pem
*.key
*.cert
*.p12
*.pfx
secrets/
credentials/
terraform.tfvars
terraform.tfvars.json
*.tfstate
*.tfstate.backup
kubeconfig
.kube/
ansible-vault-password
```

**Best practices cho .gitignore:**

```bash
# Nếu đã lỡ commit một file, xóa khỏi tracking mà không xóa file
git rm --cached .env
git rm --cached -r node_modules/

# Xem tại sao một file bị ignore
git check-ignore -v src/secret.key

# Xem tất cả files đang bị ignore
git status --ignored

# Ignore tạm thời (chỉ cho máy của bạn, không commit)
# Thêm vào .git/info/exclude (không được version controlled)
echo "my-personal-notes.txt" >> .git/info/exclude

# Global gitignore (áp dụng cho tất cả repos trên máy bạn)
git config --global core.excludesfile ~/.gitignore_global
```

---

### 8. Semantic Commit Messages

Conventional Commits là một specification cho việc viết commit messages có cấu trúc. Không chỉ là convention đẹp — nó enable automation.

#### Format

```
type(scope): description

[optional body]

[optional footer(s)]
```

**Quy tắc:**
- `type`: bắt buộc, lowercase
- `scope`: optional, chỉ rõ phần nào của codebase bị ảnh hưởng
- `description`: bắt buộc, lowercase, không dấu chấm cuối, tối đa 72 ký tự
- Body: ngăn cách bởi dòng trống, giải thích WHY (không phải WHAT)
- Footer: Breaking changes, issue references

#### Các types

```
feat     — Tính năng mới (MINOR version bump trong SemVer)
fix      — Sửa bug (PATCH version bump)
docs     — Chỉ thay đổi documentation
style    — Formatting, whitespace, không thay đổi logic (không phải CSS style)
refactor — Tái cấu trúc code, không fix bug, không thêm feature
test     — Thêm hoặc sửa tests
chore    — Cập nhật dependencies, build scripts, config files
ci       — Thay đổi CI/CD configuration
perf     — Cải thiện performance
build    — Ảnh hưởng đến build system hoặc external dependencies
revert   — Revert một commit trước đó
```

**Breaking Change** (MAJOR version bump):
Thêm `!` sau type/scope: `feat(api)!:` hoặc thêm `BREAKING CHANGE:` trong footer.

#### Ví dụ thực tế

```bash
# Feature đơn giản
git commit -m "feat(auth): add Google OAuth login"

# Bug fix với issue reference
git commit -m "fix(cart): prevent duplicate items when clicking rapidly

User reported that clicking 'Add to Cart' button quickly multiple times
could result in duplicate items. Added debounce of 300ms to the handler.

Fixes #123"

# Breaking change
git commit -m "feat(api)!: change /products endpoint to return paginated response

BREAKING CHANGE: /api/products now returns {data: [], meta: {pagination}} 
instead of a plain array. Update all API consumers accordingly."

# Chore
git commit -m "chore(deps): upgrade express from 4.17.1 to 4.18.2"

# CI/CD change
git commit -m "ci(github-actions): add Docker build layer caching

Reduces CI build time from ~8 minutes to ~2 minutes by caching
Docker layers between runs."

# Docs
git commit -m "docs(readme): add local development setup instructions"

# Performance
git commit -m "perf(db): add index on products.category_id column

Products search by category was doing full table scan.
New index reduces query time from 2.3s to 12ms on 100k records."

# Revert
git commit -m "revert: feat(auth): add Google OAuth login

This reverts commit abc1234.
Reverting due to security vulnerability in google-auth library v2.3.1.
Will re-enable after library is patched."
```

#### Tại Sao Semantic Commits Quan Trọng

1. **Automated CHANGELOG generation**: Tools như `conventional-changelog` hay `semantic-release` đọc commit history và tự động tạo CHANGELOG.md.

2. **Automated versioning**: `semantic-release` tự động bump version number dựa trên commit types (feat → minor, fix → patch, BREAKING CHANGE → major).

3. **Better `git log`**: Dễ scan và filter lịch sử.

4. **Communication**: Reviewer hiểu intent của PR ngay từ commit messages.

5. **GitHub Release notes**: Tự động categorize commits theo type.

---

### 9. GitHub Workflow Thực Tế

#### Branch Protection Rules cho `main`

Đây là cấu hình bắt buộc cho bất kỳ project nghiêm túc nào. Vào: GitHub Repo → Settings → Branches → Add rule → Pattern: `main`

```
Cài đặt bắt buộc:
[x] Require a pull request before merging
    [x] Require approvals: 1 (solo dev: 0, nhưng vẫn require PR)
    [x] Dismiss stale pull request approvals when new commits are pushed
    [x] Require review from Code Owners (nếu có CODEOWNERS file)
    
[x] Require status checks to pass before merging
    [x] Require branches to be up to date before merging
    Add status checks: build, test, lint (từ CI pipeline của bạn)

[x] Require conversation resolution before merging
    (tất cả review comments phải được addressed)

[x] Do not allow bypassing the above settings
    (áp dụng cả cho administrators — bao gồm chính bạn)

Recommended:
[x] Require linear history (no merge commits, enforce squash/rebase)
[ ] Include administrators (cẩn thận: bạn cũng không thể bypass)
```

#### Pull Request Template

Tạo file `.github/pull_request_template.md` trong repo:

```markdown
## Tóm tắt thay đổi
<!-- Mô tả ngắn gọn những gì bạn đã thay đổi -->

## Lý do thay đổi
<!-- Tại sao cần thay đổi này? Giải quyết vấn đề gì? Link issue nếu có. -->
Closes #

## Loại thay đổi
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change
- [ ] Documentation update
- [ ] Refactoring (no functional changes)

## Cách test
<!-- Mô tả các bước để verify thay đổi này hoạt động đúng -->
1. 
2. 
3. 

## Screenshots (nếu có thay đổi UI)
<!-- Thêm screenshot before/after nếu có -->

## Checklist
- [ ] Code follow coding standards của project
- [ ] Self-review code của mình trước khi submit
- [ ] Viết/cập nhật tests cho thay đổi này
- [ ] Tests pass locally
- [ ] Không có secrets, credentials, hay PII trong code
- [ ] Documentation được cập nhật (nếu cần)
```

#### CODEOWNERS File

File `.github/CODEOWNERS` tự động assign reviewer dựa trên thư mục/file:

```
# Mặc định: mọi PR cần review từ team leads
*                   @shoplite/leads

# Backend code cần backend team review
/src/api/           @shoplite/backend
/src/models/        @shoplite/backend
/src/controllers/   @shoplite/backend

# Frontend code cần frontend team review
/src/components/    @shoplite/frontend
/src/pages/         @shoplite/frontend

# Infrastructure cần DevOps review
/terraform/         @shoplite/devops
/k8s/               @shoplite/devops
/.github/           @shoplite/devops
Dockerfile          @shoplite/devops
docker-compose.yml  @shoplite/devops

# Security-sensitive files — cần cả team leads và devops
/src/auth/          @shoplite/leads @shoplite/devops
```

#### Code Review Etiquette

**Cho Reviewer:**

```
Nên làm:
- Comment cụ thể: "Dòng 45: ..." thay vì "code này sai"
- Explain why: "Nên dùng Set thay vì Array ở đây vì lookup O(1) thay vì O(n)"  
- Suggest alternative: "Có thể viết ngắn hơn thành: const result = items.find(...);"
- Phân biệt blocking vs non-blocking:
  - "nit: ..." = nitpick, non-blocking, author có thể ignore
  - "suggestion: ..." = improvement idea, non-blocking
  - "[blocking] ..." = phải fix trước khi merge
- Acknowledge good code: "Nice solution here!"
- Ask questions: "Tại sao chọn approach này thay vì X?"

Không nên:
- "Code này tệ" (không cụ thể)
- Chỉ comment về style thay vì logic
- Approve mà không thực sự đọc code
- Xem code review là cơ hội để thể hiện kiến thức
```

**Cho Author:**

```
Nên làm:
- Self-review trước khi submit (đọc lại diff của chính mình)
- Viết PR description rõ ràng để reviewer có context
- Respond tất cả comments, kể cả khi không đồng ý
- Khi không đồng ý: explain reasoning, không phải defensive
- "Đã fix" hoặc "Done" sau khi address comment
- Mark conversations as "resolved" sau khi fixed

Không nên:
- Submit PR với 50 files changed mà không explanation
- Ignore comments
- Force merge dù chưa có approval
```

#### GitHub Actions CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Run tests
        run: npm test -- --coverage
      
      - name: Build
        run: npm run build
```

#### Tags và Releases

```bash
# Tạo annotated tag cho release (khuyến nghị)
git tag -a v1.0.0 -m "Release version 1.0.0 - Initial production release"

# Tạo tag tại một commit cụ thể
git tag -a v1.0.0 abc1234 -m "Release version 1.0.0"

# Push tag lên remote
git push origin v1.0.0
git push origin --tags  # push tất cả tags

# Xem danh sách tags
git tag
git tag -l "v1.*"  # filter theo pattern

# Xem chi tiết một tag
git show v1.0.0

# Xóa tag local
git tag -d v1.0.0

# Xóa tag trên remote
git push origin --delete v1.0.0
```

---

### 10. CONTRIBUTING.md Template cho ShopLite

Tạo file `CONTRIBUTING.md` ở root của repo:

```markdown
# Contributing to ShopLite

Cảm ơn bạn đã muốn đóng góp cho ShopLite! Tài liệu này mô tả quy trình làm việc
của team để giữ codebase nhất quán và chất lượng cao.

## Mục lục

1. [Setup môi trường phát triển](#setup-môi-trường-phát-triển)
2. [Branching Strategy](#branching-strategy)
3. [Commit Message Convention](#commit-message-convention)
4. [Pull Request Process](#pull-request-process)
5. [Code Review Guidelines](#code-review-guidelines)
6. [Running Tests](#running-tests)

---

## Setup Môi Trường Phát Triển

### Yêu cầu

- Node.js >= 20.x
- npm >= 10.x
- Docker Desktop (cho local database)
- Git >= 2.39

### Các bước setup

1. Fork repo và clone về máy:
   
   git clone https://github.com/YOUR_USERNAME/shoplite.git
   cd shoplite

2. Cài đặt dependencies:
   
   npm install

3. Copy environment template và điền values:
   
   cp .env.example .env.local
   # Mở .env.local và điền các giá trị cần thiết

4. Khởi động database:
   
   docker-compose up -d postgres redis

5. Chạy database migrations:
   
   npm run db:migrate

6. Seed database với test data:
   
   npm run db:seed

7. Khởi động development server:
   
   npm run dev

8. Truy cập http://localhost:3000

---

## Branching Strategy

ShopLite sử dụng **GitHub Flow**:

- `main` là branch duy nhất tồn tại lâu dài
- `main` luôn trong trạng thái có thể deploy lên production
- Mọi thay đổi đều phải đi qua Pull Request

### Quy ước đặt tên branch

```
feature/  — Tính năng mới
fix/      — Bug fixes  
docs/     — Documentation
refactor/ — Code refactoring
test/     — Tests
ci/       — CI/CD changes
chore/    — Maintenance tasks
```

Ví dụ:
- `feature/product-search-filters`
- `fix/cart-duplicate-items`
- `docs/api-authentication`

### Workflow

```bash
# 1. Cập nhật main trước khi bắt đầu
git switch main
git pull origin main

# 2. Tạo branch mới
git switch -c feature/your-feature-name

# 3. Commit thường xuyên với semantic messages
git commit -m "feat(scope): description"

# 4. Push lên remote ngay từ đầu (để backup và visibility)
git push -u origin feature/your-feature-name

# 5. Tạo Pull Request khi ready for review
# 6. Address feedback và push thêm commits
# 7. Sau khi approved và CI pass → Merge PR
# 8. Xóa feature branch
```

---

## Commit Message Convention

ShopLite tuân theo **Conventional Commits specification**.

### Format

```
type(scope): description

[optional body]

[optional footer]
```

### Types

| Type | Mô tả |
|------|-------|
| `feat` | Tính năng mới |
| `fix` | Sửa bug |
| `docs` | Chỉ thay đổi documentation |
| `style` | Formatting (không thay đổi logic) |
| `refactor` | Refactor (không fix bug, không thêm feature) |
| `test` | Thêm hoặc sửa tests |
| `chore` | Build process, dependencies |
| `ci` | CI/CD changes |
| `perf` | Cải thiện performance |

### Rules

- Subject dòng đầu: tối đa 72 ký tự
- Dùng imperative, present tense: "add feature" chứ không phải "added feature"
- Không viết hoa chữ cái đầu của description
- Không dấu chấm cuối dòng subject
- Body: giải thích WHY, không phải WHAT

### Ví dụ

```bash
feat(auth): add JWT refresh token rotation
fix(cart): prevent duplicate items on rapid clicks
docs(api): update authentication endpoint documentation
ci: add Docker build cache to reduce CI time
perf(db): add index on products.category_id
```

### Breaking Changes

```bash
feat(api)!: change products endpoint response structure

BREAKING CHANGE: /api/products now returns paginated response.
Migration guide: https://docs.shoplite.com/migration/v2
```

---

## Pull Request Process

### Trước khi tạo PR

- [ ] Self-review toàn bộ diff của mình
- [ ] Tests pass: `npm test`
- [ ] Lint pass: `npm run lint`
- [ ] Build thành công: `npm run build`
- [ ] Không có secrets hay credentials trong code
- [ ] PR title follow Conventional Commits format

### Tạo PR

1. Dùng PR template có sẵn
2. Title: ngắn gọn, mô tả thay đổi chính
3. Description: giải thích context, link related issues
4. Assign reviewer phù hợp
5. Add labels (enhancement, bug, documentation, etc.)

### Merge Policy

- Cần ít nhất 1 approval (2 approvals cho changes lớn)
- Tất cả CI checks phải pass
- Tất cả review conversations phải được resolved
- Branch phải up-to-date với main trước khi merge
- Dùng **Squash and merge** cho feature branches (giữ main history clean)
- Dùng **Create a merge commit** cho release merges (preserve release history)

---

## Code Review Guidelines

### Cho Reviewer

**Làm:**
- Comment cụ thể với context và suggestion
- Phân biệt blocking (`[blocking]`) và non-blocking (`nit:`, `suggestion:`)
- Explain reasoning của feedback
- Acknowledge good implementations

**Không làm:**
- Comment về style issues đã được handle bởi linter
- Request changes mà không explain tại sao
- Approve mà không thực sự đọc code

### Cho Author

**Làm:**
- Respond tất cả comments
- Khi không đồng ý: explain reasoning
- Mark conversations as resolved sau khi addressed
- Cảm ơn reviewer cho feedback hữu ích

**Không làm:**
- Submit PR với hundreds of files changed
- Merge trước khi có approval
- Ignore review comments

---

## Running Tests

### Unit Tests

```bash
# Chạy tất cả tests
npm test

# Watch mode (auto re-run khi có thay đổi)
npm run test:watch

# Với coverage report
npm run test:coverage

# Chỉ chạy tests matching pattern
npm test -- --testPathPattern="auth"
```

### Integration Tests

```bash
# Yêu cầu: database đang chạy
npm run test:integration
```

### E2E Tests

```bash
# Yêu cầu: full application đang chạy
npm run test:e2e
```

### Viết Tests

- Unit test file đặt cạnh file implementation: `auth.js` → `auth.test.js`
- Integration tests: `tests/integration/`
- Test naming: `describe('AuthService', () => { it('should return user when credentials valid', ...)`
- Coverage threshold: minimum 80% cho statements và branches

---

## Liên Hệ

Nếu có câu hỏi, mở GitHub Discussion hoặc liên hệ maintainers qua email trong README.
```

---

### 11. Git Workflow Nâng Cao

#### Bisect — Tìm Bug Bằng Binary Search

```bash
# Khi bạn biết bug tồn tại ở commit hiện tại nhưng không biết commit nào gây ra

# Bắt đầu bisect session
git bisect start

# Đánh dấu commit hiện tại là "bad" (có bug)
git bisect bad

# Đánh dấu commit cũ mà bạn biết "good" (không có bug)
git bisect good v1.0.0

# Git checkout commit ở giữa, bạn test và đánh dấu
git bisect good  # hoặc
git bisect bad

# Lặp lại cho đến khi Git tìm ra commit gây bug
# Git sẽ báo: "abc1234 is the first bad commit"

# Kết thúc bisect, quay về HEAD
git bisect reset

# Bisect tự động với script test
git bisect run npm test
```

#### Submodules

```bash
# Thêm repo khác như submodule (thường cho shared libraries)
git submodule add https://github.com/shoplite/ui-components.git src/ui

# Clone repo có submodules
git clone --recursive https://github.com/user/shoplite.git

# Update submodules sau khi pull
git submodule update --init --recursive

# Pull updates cho tất cả submodules
git submodule update --remote
```

#### Worktree

```bash
# Làm việc trên nhiều branches đồng thời không cần stash
# Checkout branch vào một thư mục khác trong khi vẫn giữ thư mục hiện tại

git worktree add ../shoplite-hotfix hotfix/critical-bug

# Xem danh sách worktrees
git worktree list

# Xóa worktree
git worktree remove ../shoplite-hotfix
```

---

## Kỹ năng đạt được
- Quản lý source code theo nhánh, xử lý conflict.
- Tạo và review Pull Request.
- Tổ chức repo theo chuẩn dự án thật.

---

## Thực hành

**Môi trường:** GitHub (hoặc GitLab) + máy local.

**Lab:**
- Đưa toàn bộ source ShopLite lên Git, viết `.gitignore` chuẩn.
- Thiết lập cấu trúc nhánh: `main`, `develop`, `feature/*`.
- Thực hành tạo feature branch → PR → review → merge.
- Tạo cố tình một conflict và giải quyết.

**Công cụ:** Git, GitHub/GitLab.

---

## Bài tập

- **Bắt buộc:** Tổ chức repo ShopLite với README, .gitignore, cấu trúc thư mục rõ ràng.
- **Nâng cao:** Cấu hình branch protection cho `main` (bắt buộc PR + review).
- **Mô phỏng doanh nghiệp:** Viết file `CONTRIBUTING.md` mô tả branching strategy và quy ước commit của team.

---

## Deliverable
- [ ] Repo ShopLite hoàn chỉnh trên GitHub/GitLab.
- [ ] Branching strategy được áp dụng + branch protection.
- [ ] `README.md`, `.gitignore`, `CONTRIBUTING.md`.

---

## Tiêu chí hoàn thành
- [ ] Toàn bộ source nằm trên Git với lịch sử commit sạch.
- [ ] Thực hiện trọn vẹn 1 vòng feature branch → PR → merge.
- [ ] Giải thích được branching strategy đang dùng và vì sao.

---

## Ghi chú học tập

> _Ghi lại những điều học được, vướng mắc và insight trong quá trình học phase này._
