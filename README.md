# Mailplug_사전과제(iOS) 이창형
 
## Tech Stack

-   **언어**  : Swift
-   **Deployment Target**: iOS 14
-   **UI**: UIKit, CodeBase(SnapKit)
-   **아키텍처**  : MVVM + Coordinator
-   **비동기 처리**  : RxSwift, RxCocoa
-   **네트워킹**  : Alamofire


## 아키텍처
### MVVM

## 주요 기능 설명
**1) 게시판 목록 화면**
✅ ViewController
```swift
    private func setTableView() {
      // viewModel을 통해서 게시글 목록을 들고옵니다. (게시판id를 넘겨주어서 게시판에 맞는 데이터들을 들고옵니다)
        viewModel.getPostData(id: boardID, offset: 0, limit: 0)

        // viewModel의 postObservable은 getPostData를 통해서 가져온 데이터들로 변경이 되는 Relay입니다
        viewModel.postObservable
            .observe(on: MainScheduler.instance)
            // postObservable의 데이터를 tableView로 바인딩 하여 화면에 나타내 줍니다.
            .bind(to: tableView.rx.items(cellIdentifier: PostTableViewCell.identifier, cellType: PostTableViewCell.self)) { index, item, cell in
                
                cell.data = item
                self.searchTextField.placeholder = "\(self.titleLabel.text!)에서 검색"
                
            }
            .disposed(by: disposeBag)
        
        tableView.rx
            .setDelegate(self)
            .disposed(by: disposeBag)
    }
    ```

✅ ViewModel
```swift
lazy var postObservable = BehaviorRelay<[PostRequestValue]>(value: [])

// 미리 만들어 놓은 APIHelpers의 getPosts를 통해서 데이터들을 들고와서 postObservable에 넣어줍니다
 func getPostData(id: Int, offset: Int, limit: Int) {
        APIHelpers.shared.getPosts(boards_id: id, offset: offset, limit: limit)
            .map { model -> [PostRequestValue] in
                return model.value
            }
            .bind(to: postObservable)
            .disposed(by: disposeBag)
    }
```

**2) 게시판 이동**
✅ ViewController (모달로 띄운 뷰)
```swift
 private func setTableView() {
       // getBoardsData를 통해 어떤 게시판들의 데이터를 들고옵니다
        viewModel.getBoardsData()
        // viewModel에서 getBoardsData를 바인딩하고 있는 boardObservable을 통해 데이터를 받아오고 tableView에 나타낼 정보를 뿌려줍니다
        viewModel.boardObservable
            .observe(on: MainScheduler.instance)
            .bind(to: tableView.rx.items(cellIdentifier: BoardTableViewCell.identifier, cellType: BoardTableViewCell.self)) { index, item, cell in
                cell.data = item
            }
            .disposed(by: disposeBag)
        
        
        tableView.rx
            .setDelegate(self)
            .disposed(by: disposeBag)
        
        // 셀 터치 이벤트 처리
        tableView.rx.itemSelected
            .subscribe(onNext: { [weak self] indexPath in
                // 선택된 셀의 데이터를 가져와서 처리
                if let selectedData = self?.viewModel.boardObservable.value[indexPath.row] {
                   // 게시판 뷰를 재사용
                    let noticeVC = NoticeBoardViewController()
                    // 게시판이 어떤 게시판인지 알려주기 위해 boardID를 넘겨줌
                    noticeVC.boardID = selectedData.boardID
                    noticeVC.modalPresentationStyle = .fullScreen
                    noticeVC.titleLabel.text = selectedData.displayName
                    
                    self?.present(noticeVC, animated: true, completion: nil)
                }
                
                
            })
            .disposed(by: disposeBag)
    }
```

✅ ViewModel
```swift
// 데이터를 들고오는 것을 bind
lazy var boardObservable = BehaviorRelay<[BoardRequestValue]>(value: [])

func getBoardsData() {
        APIHelpers.shared.getBoards()
            .map{ model -> [BoardRequestValue] in
                return model.value
            }
            .bind(to: boardObservable)
            .disposed(by: disposeBag)
    }
```

**3) 검색화면**
✅ ViewController
```swift

// 검색중 표시되는 tableView
    private let searchTableView: UITableView = {
        let tableView = UITableView()
        tableView.register(SearchTableViewCell.self, forCellReuseIdentifier: SearchTableViewCell.identifier)
        tableView.separatorStyle = .none
        tableView.isHidden = true
        
        return tableView
    }()

// 검색 결과 tableView
    private let searchResultTableView: UITableView = {
        let tableView = UITableView()
        tableView.register(PostTableViewCell.self, forCellReuseIdentifier: PostTableViewCell.identifier)
        tableView.separatorStyle = .none
        tableView.isHidden = true
        
        return tableView
    }()

private func setSearchTableView() {
        searchItems
            .bind(to: searchTableView.rx.items(cellIdentifier: SearchTableViewCell.identifier, cellType: SearchTableViewCell.self)) { [self] (row, element, cell) in
                cell.searchTargetLabel.text = element
                // searchBar 검색할때 실시간으로 검색 결과를 가져오는 코드
                viewModel.getText
                    .subscribe(onNext: {a in
                        cell.searchLabel.text = a
                    })
                cell.selectionStyle = .none
            }
            .disposed(by: disposeBag)
        
        // searchTableView의 특정 셀을 선택했을 때의 이벤트 처리
        searchTableView.rx.itemSelected
            .subscribe(onNext: { [weak self] indexPath in
                guard let cell = self?.searchTableView.cellForRow(at: indexPath) as? SearchTableViewCell,
                      let searchTargetLabel = cell.searchTargetLabel.text else {
                    // 셀이 선택되지 않은 경우 또는 searchTargetLabel이 nil인 경우
                    return
                }
                guard let searchTextFieldText = self?.searchTextField.text else {
                                    return
                                }
                
                
                print("Selected Target: \(searchTargetLabel)")
                var target = ""

              // API통신을 위해 데이터 변경
                switch searchTargetLabel {
                case "전체":
                    target = "all"
                case "제목":
                    target = "title"
                case "내용":
                    target = "contents"
                case "작성자":
                    target = "writer"
                default:
                    break
                }
                
                guard let searchTextFieldText = self?.searchTextField.text else {
                    // searchTextFieldText가 nil인 경우
                    return
                }
                
                // 여기에 검색 데이터를 가져오는 로직 추가
                self?.viewModel.getSearchsData(board_id: self!.boardID, search: searchTextFieldText, searchTarget: target, offset: 0, limit: 0)
                
                if self?.viewModel.searchObservable.value.count == 0 {
                    self?.searchTableView.isHidden = true
                    self?.searchResultImageView.isHidden = false
                    self?.resultHelpSearchLabel.isHidden = false
                    
                } else {
                    self?.searchResultTableView.isHidden = false
                }
            })
            .disposed(by: disposeBag)
    }

  // 검색 결과 화면
    private func setSearchResultTableView() {
        searchResultTableView.rowHeight = 74
        // searchObservable은 getSearchsData(검색결과 가져오는 함수)를 bind하고 있습니다
        viewModel.searchObservable
            .observe(on: MainScheduler.instance)
            // searchObservable를 bind해서 tableView에 표시해줍니다
            .bind(to: searchResultTableView.rx.items(cellIdentifier: PostTableViewCell.identifier, cellType: PostTableViewCell.self)) { index, item, cell in
                cell.resultData = item
            }
    }
```

✅ ViewModel
```swift
let searchTextFieldObserver = BehaviorRelay<String>(value: "")

// textfield가 변경될때마다 searchTextFieldObserver에 저장
 var getText: Observable<String> {
        return searchTextFieldObserver.asObservable()
            .map { text in
        
                return text
            }
    }

// 검색결과를 bind
lazy var searchObservable = BehaviorRelay<[SearchRequestValue]>(value: [


    func getSearchsData(board_id: Int, search: String, searchTarget: String, offset: Int, limit: Int) {
        APIHelpers.shared.getSearchs(board_id: board_id, search: search, searchTarget: searchTarget, offset: offset, limit: limit)
            .map{ model -> [SearchRequestValue] in
                return model.value
            }
            .bind(to: searchObservable)
            .disposed(by: disposeBag)
    }

```
