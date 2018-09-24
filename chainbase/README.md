

# chainbase

chainbase依赖于boost.interprocess，进而依赖nmap实现文件到内存映射，可以像操作内存一样对文件进行随机存储，nmap负责
将内容同步到文件中

chainbase::database中每个表由MultiIndex（使用static const uint16_t type_id标识)表示，每个表存储一种类型的对象，
每个对象使用oid<Object> id 作为主键

chainbase使用状态栈保存数据的改动，

#
<span style="color:pink">
以下代码：定义构造器内存分配器，用于在共享内存里创建对象
</span>

```cpp
template<typename T>
using allocator = bip::allocator<T, bip::managed_mapped_file::segment_manager>;
```

#
<span style="color:pink">
以下代码：定义使用共享内存构造的string和vector
</span>

```cpp
typedef bip::basic_string< char, std::char_traits< char >, allocator< char > > shared_string;

template<typename T>
using shared_vector = std::vector<T, allocator<T> >;
```

#
<span style="color:pink">
以下代码：标记数据的dirty_flag
</span>

```cpp
constexpr char _db_dirty_flag_string[] = "db_dirty_flag";
```


#
<span style="color:pink">
以下代码：共享内存互斥保护
</span>

```cpp
typedef boost::interprocess::interprocess_sharable_mutex read_write_mutex;
typedef boost::interprocess::sharable_lock< read_write_mutex > read_lock;
typedef boost::unique_lock< read_write_mutex > write_lock;
```

#
<span style="color:pink">
以下代码：定义用于标记数据对象id的类
</span>

```cpp
template<typename T>
class oid {
    oid( int64_t i = 0 ):_id(i){}
    ...
    int64_t _id
}
```

#
<span style="color:pink">
以下代码：定义数据基类
</span>

```cpp
template<uint16_t TypeNumber, typename Derived>
struct object
{
    typedef oid<Derived> id_type;                // 定义本模版实例的对象id类型
    static const uint16_t type_id = TypeNumber;  // 定义本模版实例的类型id
};
```

#
<span style="color:pink">
以下代码：用于关联INDEX_TYPE和OBJECT_TYPE，get_index_type<OBJECT_TYPE>::type 为INDEX_TYPE
</span>

```cpp
template<typename T>
struct get_index_type {};

#define CHAINBASE_SET_INDEX_TYPE( OBJECT_TYPE, INDEX_TYPE )  \
namespace chainbase { template<> struct get_index_type<OBJECT_TYPE> { typedef INDEX_TYPE type; }; }
```

#
<span style="color:pink">
以下代码：使用模版定义构造函数
</span>

```cpp
#define CHAINBASE_DEFAULT_CONSTRUCTOR( OBJECT_TYPE ) \
template<typename Constructor, typename Allocator> \
OBJECT_TYPE( Constructor&& c, Allocator&&  ) { c(*this); }
```

#
<span style="color:pink">
以下代码：定义数据操作（添加，修改，删除）undo缓存
</span>

```cpp
template< typename value_type >
class undo_state
{
    public:
        // 数据对象object (value_type类型) 的id_type: oid<value_type> 由int64_t值表示，可隐式构造
        typedef typename value_type::id_type                      id_type;
        // 类型 pair<const id_type, value_type> (<数据对象id, 数据对象> 对) 的 allocator
        typedef allocator< std::pair<const id_type, value_type> > id_value_allocator_type;
        // 数据对象id (id_type) 的 allocator
        typedef allocator< id_type >                              id_allocator_type;

        template<typename T>
        undo_state( allocator<T> al )
        :old_values( id_value_allocator_type( al.get_segment_manager() ) ),
        removed_values( id_value_allocator_type( al.get_segment_manager() ) ),
        new_ids( id_allocator_type( al.get_segment_manager() ) ){}

        // 数据对象id (id_type) 到 value (value_type) 的 map
        typedef boost::interprocess::map< id_type, value_type, std::less<id_type>, id_value_allocator_type >  id_value_type_map;
        // 数据对象id (id_type) 的set
        typedef boost::interprocess::set< id_type, std::less<id_type>, id_allocator_type >                    id_type_set;

        id_value_type_map            old_values;
        id_value_type_map            removed_values;
        id_type_set                  new_ids;
        id_type                      old_next_id = 0;
        int64_t                      revision = 0;
};
```

#
<span style="color:pink">
以下代码：定义managed_mapped_file中索引表的操作类
</span>

```cpp
template<typename MultiIndexType>
class generic_index
{
    // --------------------------------
    // --- 类型定义
    // --------------------------------
    // 文件内存映射的segment管理器
    typedef bip::managed_mapped_file::segment_manager             segment_manager_type;
    // index表类型
    typedef MultiIndexType                                        index_type;
    // index表中数据对象类型
    typedef typename index_type::value_type                       value_type;
    // index表构造器内存分配器
    typedef bip::allocator< generic_index, segment_manager_type > allocator_type;
    // 数据对象的undo_state缓存
    typedef undo_state< value_type >                              undo_state_type;

    // --------------------------------
    // --- 成员
    // --------------------------------
    // 数据修改缓冲栈
    boost::interprocess::deque< undo_state_type, allocator<undo_state_type> > _stack;
    // 表当前的revision
    int64_t                         _revision = 0;
    // 添加数据对象时可用的id
    typename value_type::id_type    _next_id = 0;
    // 存储数据对象的多索引表
    index_type                      _indices;
    // 用于类型检查验证
    uint32_t                        _size_of_value_type = 0;
    uint32_t                        _size_of_this = 0;

    // --------------------------------
    // --- 操作
    // --------------------------------
    // 构造函数，指定内存分配器，并保存类型大小用于验证
    generic_index( allocator<value_type> a )
    :_stack(a),_indices( a ),_size_of_value_type( sizeof(typename MultiIndexType::node_type) ),_size_of_this(sizeof(*this)){}

    // 类型验证检查
    void validate()const {
    if( sizeof(typename MultiIndexType::node_type) != _size_of_value_type || sizeof(*this) != _size_of_this )
        BOOST_THROW_EXCEPTION( std::runtime_error("content of memory does not match data expected by executable") );
    }

    // 表中创建一个数据对象，并赋于对象唯一id，最后返回对象引用，如果创建数据对象错误会异常
    // 成功创建后会根据有无undo栈，保存此对象到栈顶undo state中
    template<typename Constructor>
    const value_type& emplace( Constructor&& c ) { ... }

    // 修改表中对象，如果修改对象id与表中其他对象id冲突则异常
    // 修改前会先在undo state栈中保存原来的对象
    template<typename Modifier>
    void modify( const value_type& obj, Modifier&& m ) { ... }

    // 删除表中对象
    // 删除前会先在undo state栈中保存被删除的对象
    void remove( const value_type& obj )

    // 通过键值（oid类型）查找 对象
    template<typename CompatibleKey>
    const value_type* find( CompatibleKey&& key ) { ... }

    // 通过键值（oid类型）获取对象
    template<typename CompatibleKey>
    const value_type& get( CompatibleKey&& key ) { ... }

    // 启动一个undo session：在undo stack压栈undo state，更新revision版本号并保存到此undo state
    // undo state栈是表中成员，同时session里又引用指向表，启动undo session后，调用端可以选择保存或者放弃session中对表的操作
    session start_undo_session( bool enabled )

    // 弹栈撤销保存在栈顶的数据修改
    void undo() { ... }

    // 合并栈顶部2层的数据修改缓存
    void squash() { ... }

    // 删除栈底部从指定revision号的层以下的数据修改缓存
    void commit( int64_t revision ) { ... }

    // 清除栈全部数据修改缓存
    void undo_all() { ... }

    // 通过id删除对象，如果没有找到id的对象则异常
    // 删除对象会保存到undo栈，如果启动了undo session
    void remove_object( int64_t id ) { ... }
}
```

<span style="color:pink">
以下代码：定义generic_index::session，用于缓存generic_index的数据修改
</span>

```cpp
class generic_index{

    ...

    class session {
        // --------------------------------
        // --- 操作
        // --------------------------------
        // move构造：session结束时保留数据变动缓存
        session( session&& mv )
        :_index(mv._index),_apply(mv._apply){ mv._apply = false; }

        // session结束时根据_apply标记，保留/丢弃数据变动缓存
        ~session() {
            if( _apply ) {
                _index.undo();
            }
        }

        // 标记：保留数据变动缓存
        void push()   { _apply = false; }
        // 合并：合并栈顶2个（对应当前session与前一个session)数据变动缓存层
        void squash() { if( _apply ) _index.squash(); _apply = false; }
        // 丢弃：丢弃栈顶的数据变动缓存
        void undo()   { if( _apply ) _index.undo();  _apply = false; }

        // 赋值move，move前先根据_apply标记，保留/丢弃数据变动缓存
        session& operator = ( session&& mv ) {
            if( this == &mv ) return *this;
            if( _apply ) _index.undo();
            _apply = mv._apply;
            mv._apply = false;
            return *this;
        }

        // 构造函数
        session( generic_index& idx, int64_t revision )
        :_index(idx),_revision(revision) {
            if( revision == -1 )
                _apply = false;
        }

        // --------------------------------
        // --- 成员
        // --------------------------------
        // 对包含类generic_index对象的应用
        generic_index& _index;
        // 默认session结束时不保留数据修改缓存
        bool           _apply = true;
        // 记录session的revision号
        int64_t        _revision = 0;
    };

    ...
}
```

<span style="color:pink">
以下代码：定义数据仓库，数据仓库用于增加，查找表；以及增加，查找，修改，删除表中的数据对象，此数据仓库database的对象通过内存映射文件可实现对文件内容的任意位置访问
</span>

```cpp
class database
{
    // --------------------------------
    // --- 类型定义
    // --------------------------------
    // 读写flag
    enum open_flags {
        read_only     = 0,
        read_write    = 1
    };
    // 各个表对象数量统计
    using database_index_row_count_multiset = std::multiset<std::pair<unsigned, std::string>>;

    // --------------------------------
    // --- 成员
    // --------------------------------
    // 文件内存映射对象指针，_segment存储数据，_meta存储mutex
    unique_ptr<bip::managed_mapped_file>                        _segment;
    unique_ptr<bip::managed_mapped_file>                        _meta;
    // 读写mutex管理
    read_write_mutex_manager*                                   _rw_manager = nullptr;
    // 文件读写标记
    bool                                                        _read_only = false;
    // 写操作锁保护
    bip::file_lock                                              _flock;

    // 索引表list
    vector<abstract_index*>                                     _index_list;
    // 带索引表指针管理的list
    vector<unique_ptr<abstract_index>>                          _index_map;

    // 映射文件路径
    bfs::path                                                   _data_dir;

    // CHAINBASE_CHECK_LOCKING 宏激活时使用的变量
    int32_t                                                     _read_lock_count = 0;
    int32_t                                                     _write_lock_count = 0;
    bool                                                        _enable_require_locking = false;

    // --------------------------------
    // --- 操作
    // --------------------------------
    // 构造函数: dir 路径，write 写标记，shared_file_size 指定大小，allow_dirty 允许dirty标记为真
    // 如果文件存在则进行兼容性检查，不兼容则异常，如果dirty标记设为真并且不允许dirty则异常
    // 最后文件对象创建或打开以后设置dirty标记为真
    database(const bfs::path& dir, open_flags write = read_only, uint64_t shared_file_size = 0, bool allow_dirty = false);

    // 析构函数：同步数据到文件，同时设置dirty标记为false，然后清空表list
    ~database();

    // 同步数据到文件
    flush();

    // 启动数据仓库的修改缓存session（每个表都启动修改缓存session）
    session start_undo_session( bool enabled );

    // 对数据仓库所有表执行undo, squash, commit, undo_all
    void undo();
    void squash();
    void commit( int64_t revision );
    void undo_all()

    // 如果只从数据仓库接口调用start_undo_session，则数据仓库中所有表的revision保持一致
    int64_t revision();
    void set_revision( uint64_t revision );

    // 向仓库添加表，如果不存在，先文件映射里创建表index结构，然后再记录到_index_map和_index_list
    template<typename MultiIndexType>
    void add_index();

    // 根据模版实例返回相应的表（generic_index类型）
    template<typename MultiIndexType>
    const generic_index<MultiIndexType>& get_index()const

    // 根据模版实例返回表（MultiIndexType类型)
    template<typename MultiIndexType, typename ByIndex>
    auto get_index()const -> decltype( ((generic_index<MultiIndexType>*)( nullptr ))->indices().template get<ByIndex>() )

    // 根据模版实例（数据类型为ObjectType)，经过IndexedByType过滤返回的表，通过键值(CompatibleKey类型)查询对象
    template< typename ObjectType, typename IndexedByType, typename CompatibleKey >
    const ObjectType* find( CompatibleKey&& key )const
    // 调用上面find获取对象
    template< typename ObjectType, typename IndexedByType, typename CompatibleKey >
    const ObjectType& get( CompatibleKey&& key )const

    // 从表中通过对象id查询ObjectType类型数据
    template< typename ObjectType >
    const ObjectType* find( oid< ObjectType > key = oid< ObjectType >() ) const
    // 通过调用上面find获取对象
    template< typename ObjectType >
    const ObjectType& get( const oid< ObjectType >& key = oid< ObjectType >() )const

    // 在ObjectType类型对象表，通过Modifier类型修改对象
    template<typename ObjectType, typename Modifier>
    void modify( const ObjectType& obj, Modifier&& m )

    // 从ObjectType类型对象表里删除对象
    template<typename ObjectType>
    void remove( const ObjectType& obj )

    // 在ObjectType类型对象表里创建对象
    template<typename ObjectType, typename Constructor>
    const ObjectType& create( Constructor&& con )

    // 获取对共享内存/文件映射的读/写操作互斥锁
    template< typename Lambda >
    auto with_read_lock( Lambda&& callback, uint64_t wait_micro = 1000000 ) const -> decltype( (*(Lambda*)nullptr)() )
    template< typename Lambda >
    auto with_write_lock( Lambda&& callback, uint64_t wait_micro = 1000000 ) -> decltype( (*(Lambda*)nullptr)() )
}
```

```cpp
class database
{
    ...

    struct session
    {
        // --------------------------------
        // --- 成员
        // --------------------------------
        // 所有表的session
        vector< std::unique_ptr<abstract_session> > _index_sessions;
        // 所有表的session有相同的revision
        int64_t _revision = -1;

        // move构造
        session( session&& s ):_index_sessions( std::move(s._index_sessions) ),_revision( s._revision ){}
        session( vector<std::unique_ptr<abstract_session>>&& s ):_index_sessions( std::move(s) )
        // 析构
        ~session()

        // 对所有session进行push squash undo操作
        void push()
        void squash()
        void undo()
    }

    ...
}
```