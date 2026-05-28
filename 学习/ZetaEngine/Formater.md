```cpp
template <int N, std::floating_point T>  
struct std::formatter<glm::vec<N, T>, char>  
{  
    // 使用 std::formatter<double> 来处理格式说明符  
    std::formatter<T> underlying_formatter;  
  
    constexpr auto parse(std::format_parse_context& ctx)  
    {        // 将格式说明符传递给底层的 double 格式化器  
        return underlying_formatter.parse(ctx);  
    }  
    auto format(const glm::vec<N, T>& v, std::format_context& ctx) const  
    {  
        auto out = ctx.out();  
  
        // 格式化第一行  
        for (int i = 0; i < N; ++i)  
        {            out = underlying_formatter.format(v[0], ctx);  
            if (i < N - 1)  
            {                out = std::format_to(out, ", ");  
            }        }        return out;  
    }
};  
  
template <int C, int R, std::floating_point T>  
struct std::formatter<glm::mat<C, R, T>, char>  
{  
    // 使用 std::formatter<double> 来处理格式说明符  
    std::formatter<T> underlying_formatter;  
  
    constexpr auto parse(std::format_parse_context& ctx)  
    {        // 将格式说明符传递给底层的 double 格式化器  
        return underlying_formatter.parse(ctx);  
    }  
    auto format(const glm::mat<C, R, T>& m, std::format_context& ctx) const  
    {  
        auto out = ctx.out();  
  
        for (int r = 0; r < R; ++r)  
        {            for (int c = 0; c < C; ++c)  
            {                out = underlying_formatter.format(m[c][r], ctx);  
                if (c < C - 1)  
                {                    out = std::format_to(out, ", ");  
                }            }            if (r < R - 1)  
            {                out = std::format_to(out, "\n");  
            }        }  
        return out;  
    }
};  
  
template <>  
struct std::formatter<Zeta::Transform, char>  
{  
    // 使用 std::formatter<double> 来处理格式说明符  
    std::formatter<Zeta::Vec3> underlying_formatter;  
  
    constexpr auto parse(std::format_parse_context& ctx)  
    {        // 将格式说明符传递给底层的 double 格式化器  
        return underlying_formatter.parse(ctx);  
    }  
    auto format(const Zeta::Transform& t, std::format_context& ctx) const  
    {  
        auto out = ctx.out();  
  
        // 格式化第一行  
        out = std::format_to(out, "p: ");  
        out = underlying_formatter.format(t.get_position(), ctx);  
        out = std::format_to(out, "\nr: ");  
        out = underlying_formatter.format(t.get_rotation(), ctx);  
        out = std::format_to(out, "\ns: ");  
        out = underlying_formatter.format(t.get_scale(), ctx);  
        return out;  
    }
};
```